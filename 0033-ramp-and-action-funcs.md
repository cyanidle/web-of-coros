```cpp
// Примерная организация кадра сопрограммы.
// Здесь отражены наиболее важные для понимая части
// 1. resume - указатель на функцию, 
//    которая вызывается при передаче управления сопрограмме, описывает стейт-машину.
// 2. promise - объект типа Promise
// 3. state - текущее состояние
// 4. heap_allocated - был ли фрейм при создание размещен в куче
//    или фрейм был создан на стеке вызывающей стороны
// 5. args - аргументы вызова сопрограммы
// 6. locals - сохраненные локальные переменные текущего состояния
// ...
struct coroutine_frame
{
    void (*resume)(coroutine_frame *);
    promise_type promise;
    int16_t state;
    bool heap_allocated;
    // args
    // locals
    //...
};

// 1. Создание и инициализация кадра сопрограммы. Инициация выполнения.
template<typename ReturnValue, typename ...Args>
ReturnValue Foo(Args&&... args)
{
    // 1.
    // Определяем тип Promise
    using coroutine_traits = std::coroutine_traits<ReturnValue, Args...>;
    using promise_type = typename coroutine_traits::promise_type;

    // 2.
    // Создание кадра сопрограммы. 
    // Размер кадра определяется встроенными средствами компилятора
    // и зависит от размера объекта Promise, количества и размера локальных переменных
    // и аргументов, и набора вспомогательных данных,
    // необходимых для управления состоянием сопрограммы.
    // 1. Если тип promise_type имеет статический метод
    //    get_return_object_on_allocation_failure,
    //    то вызывается версия оператора new, не генерирующая исключений
    //    и в случае неудачи вызывается метод get_return_object_on_allocation_failure,
    //    результат вызова возвращается вызывающей стороне.
    // 2. Иначе вызывается обычная версия оператора new.
    coroutine_frame* frame = nullptr;
    if constexpr (has_static_get_return_object_on_allocation_failure_v<promise_type>)
    {
        frame = reinterpret_cast<coroutine_frame*>(
            operator new(__builtin_coro_size(), std::nothrow));
        if(!frame)
            return promise_type::get_return_object_on_allocation_failure();
    }
    else
    {
        frame = reinterpret_cast<coroutine_frame*>(operator new(__builtin_coro_size()));
    }

    // 3.
    // Сохраняем переданные функции аргументы во фрейме.
    // Аргументы переданные по значению перемещаются.
    // Аргументы переданные по ссылке (lvalue и rvalue) сохраняют ссылочную семантику.
    <move-args-to-frame>

    // 4.
    // Создаем объект типа promise_type и сохраняем его во фрейме
    new(&frame->promise) create_promise<promise_type>(<frame-lvalue-args>);

    // 5.
    // Вызываем метод Promise::get_return_object().
    // Результат вычисления будет возвращен вызывающей стороне
    // при достижение первой точки остановки и передачи потока управления.
    // Результат сохраняется как локальная переменная до вызова тела функции,
    // т.к. фрейм сопрограммы может быть удален (см. оператор co_await).
    auto return_object = frame->promise.get_return_object();

    // 6.
    // Вызываем функцию описывающую стейт-машину согласно 
    // пользовательским запросам передачи управления
    // В реализации GCC, например, эти две функции называются
    // ramp-fucntion (создание и инициализация) и 
    // action-function (пользовательская стейт-машина) соответственно
    void couroutine_states(coroutine_frame*);
    couroutine_states(frame); //Первый вызов

    // 7.
    // Возвращаем результат вызывающей стороне, 
    // мы достигнем этой точки в коде только при первом вызове,
    // все последующие запросы на возобновление работы будут вызывать функцию
    // стейт-машины couroutine_states, указатель на функцию сохранен во фрейме сопрограммы.
    return return_object;
}
```


```cpp
void couroutine_states(coroutine_frame* frame)
{
    switch(frame->state)
    {
        case 0:
        ... goto resume_point_0;
        case N:
            goto resume_point_N;
        ...
    }

    co_await promise.initial_suspend();

    try
    {
        // function body
    }
    catch(...)
    {
        promise.unhandled_exception();
    }

final_suspend:
    co_await promise.final_suspend();
}
```