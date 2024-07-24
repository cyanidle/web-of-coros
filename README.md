# В паутине корутин

## Зачем

### Все мы хотим писать такой код:
```cpp
task<> tcp_echo_server(tcp::socket socket)
{
    char data[1024];
    while (true) {
        auto n = co_await socket.async_read_some(buffer(data), use_awaitable);
        co_await async_write(socket, buffer(data, n));
    }
}
```
### К сожалению или счастью приходится писать следующее:
```cpp
class EchoServer : std::enable_shared_from_this<EchoServer> {
    char buff[1024];
    tcp::socket sock;
public:
    EchoServer(tcp::socket sock) : sock(std::move(sock)) {}
    void Start() {
        read();
    }
private:
    void read() {
        async_read(sock, buffer(buff), [self = shared_from_this()](auto ec, size_t read){
            if (ec) {
                //todo!
                return; //cannot throw here
            }
            self->write(read);
        });
    }
    void write(size_t amount) {
        async_write(sock, buffer(buff, amount), [self = shared_from_this()](auto ec, size_t){
            if (ec) {
                //todo!
                return; //cannot throw here
            }
            self->read();
        });

    }
};
```
## Предыстория

Мелвин Эдвард Конвей - американский ученый-компьютерщик, программист и хакер, который придумал то, что сейчас известно как закон Конвея: "Организации, разрабатывающие системы, вынуждены создавать проекты, которые являются копиями коммуникационных структур этих организаций". 

Помимо вышеперечисленного, Конвей, пожалуй, наиболее известен благодаря разработке концепции сопрограмм. Конвей ввел термин "сопрограмма" в 1958 году и был первым, кто применил эту концепцию к программе сборки. Позже он написал основополагающую статью на тему сопрограмм под названием "Design of a Separable Transition-diagram".

> Описанные алгоритмы были проверены на компьютере, с памятью в 5000 машинных слов

> Рисунок 3 иллюстрирует суть разделимости. Вместо 
того, чтобы модули A и B взаимодействовали как сопрограммы
со связью сопрограмм между операторами записи в A
и операторами чтения в B, так что управление передается туда
и обратно один раз при каждой передаче элемента, возможно, не изменяя ничего в A или B, кроме связей чтения и
записи, иметь функцию записи в A и B. все его элементы на ленте a, чтобы
перемотать ленту назад и затем попросить B прочитать все элементы с
ленты. Таким образом, в этом смысле пара программ A и B
может работать как однопроходный или двухпроходный процессор с
только тривиальная модификация

> Тем свойством конструкции, которое делает ее пригодной для
многих сегментных конфигураций, является ее разделяемость. Программная
организация является разделимой, если она разбита на
модули обработки, которые взаимодействуют друг с другом в соответствии со
следующими ограничениями: 

1) единственная связь между модулями осуществляется в виде отдельных элементов информации (сообщений) 
2) поток каждого из этих элементов осуществляется по фиксированным односторонним путям 
3) вся программа может быть построена таким образом, что входные данные находятся в крайнем левом углу, выходные - в крайнем левом верхнем углу. крайний правый элемент и все, что находится между ними, все информационные элементы, передаваемые между модулями, имеют движение вправо.


## C++20

Представим сферический проект в вакууме на libuv/Qt или
любом другом фреймворке/библиотеке с асинхроным eventloop.

Чаще всего для индикации завершения операции в высокоуровневых асинхронных рантаймах используется callback функция. Детали не так важны, передается ли некий `void* data` или используется полноценное стирание типов на плюсах или это простой `std::function<>`

И такой код может вполне быть успешным полезным. Я сам считаю что это оптимальный вариант при написании чего то низкоуровневого. Но при написании именно бизнес-логики, а не инфраструктуры иногда удобней использовать модель `async\await`. Самые болезненные случаи будут рассмотрены дальше

//TODO: Добавить обоснования из пейпера Корутин ТС

Для того чтобы начать понимание того, как все таки использовать корутины, а также интегрировать чужие начнем с простого - создадим обертку для следующей функции.

```cpp
void run(string request, std::function<void(bool ok, string result)> cb);

??? run_async(string request) {
    ???
}
```


Стандарт С++20 позволяет определить `coroutine_type` - объект, который являсь значением, возвращаемым из функции определеить ее как корутину.

Для этого **обязательно**:
1) присутствие одного из ключевых слов: `co_await`, `co_return`, `co_yield`.
2) наличие внутреннего типа `promise_type` (или переопределение `std::coroutine_traits`)
3) возможность получить из `coroutine_type` => `awaitable`

Во первых это очень сжатые требования. Во вторых когда открываешь их самостоятельно помимо них видишь на `cppreference`
```cpp
template<typename T>
struct task {
    struct promise_type {
        coroutine get_return_object() { return {coroutine::from_promise(*this)}; }
        std::suspend_always initial_suspend() noexcept { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }

        void return_value(T&& result) {} // Либо эта версия
        void return_void() {} // Либо эта

        void unhandled_exception() {} // вызывается внутри catch() блока.
    }
}
```

// TODO: Концепт awaitable

## Создаем свою имплементацию

### Представим, что мы имеем следующее асинхронное Апи

Что-то в духе Redis
```cpp
// ok == false => result contains exception msg
void run(string request, std::function<void(bool ok, string result)> cb);
```

### Силой 20-х плюсов и священного комитета сварим следующий сниппет
```cpp
#include <string>
#include <functional>
#include <coroutine>
#include <stdexcept>
#include <optional>
#include <concepts>
#include <memory>

using std::string;
```

```cpp
void run(string request, std::function<void(bool ok, string result)> cb);
void log(string msg);
```
Напишем наши корутины следующим образом: все состояние одной асинхронной транзакции будет общим 
между `promise` (передающая сторона) и `task` (принимающая сторона)
```cpp
struct state {
    bool done = false;
    std::exception_ptr exc;
    string result;
    std::coroutine_handle<> handle = {}; //type-erased
};
```
Для начала мы напишем `promise`. Фактически `handle` для передачи принимающей стороне результата асинхронного вычисления
```cpp
struct promise {
    std::shared_ptr<state> s = std::make_shared<state>();
    auto initial_suspend() {
        return std::suspend_never{};
    }
    auto final_suspend() noexcept {
        return std::suspend_never{};
    }
    auto get_return_object() {
        return task{s};
    }
    void set_error(std::exception_ptr exc) {
        s->exc = exc;
        s->done = true;
        if (s->handle) {
            s->handle.resume();
        }
    }
    auto unhandled_exception() {
        set_error(std::current_exception());
    }
    void return_value(string result) {
        s->result = std::move(result);
        s->done = true;
        if (s->handle) {
            s->handle.resume();
        }
    }
};
```
```cpp
struct task {
    std::shared_ptr<state> s;
    using promise_type = promise;
    bool await_ready() {
        return s->done;
    }
    // Хэндл вызывающей стороны!
    // Параметр шаблона может отличаться!
    // Поэтому используем стертую версию
    void await_suspend(std::coroutine_handle<> h) {
        s->handle = h;
    }
    string await_resume() {
        if (s->exc) {
            std::rethrow_exception(s->exc);
        } else {
            return std::move(s->result);
        }
    }
};
```
```cpp
// Клей со старым кодом!
task async_run(string request) {
    promise prom;
    auto future = prom.get_return_object();
    // Стоит обратить внимание, что promise_type у нас получился копируемым, что
    // обычно делать не стоит!
    run(request, [prom](auto ok, string result){
        if (ok) {
            prom.return_value(std::move(result));
        } else {
            prom.set_error(std::make_exception_ptr(std::runtime_error(std::move(result))));
        }
    });
    return future;
}
```
Ну и наконец!
```cpp
task async_main() {
    while (true) {
        auto pong = co_await async_run("ping");
        log(pong);
    }
}
```

### Возьмем теперь что-то более реалистичное!
```cpp

typedef void(*Callback)(void* data, const char* responce, bool ok); // Нам даже не пожалели typedef!
void run(const char* request, Callback callback, void* data);

```
Теперь `async_run` будет выглядеть следующим образом
```cpp
namespace detail {

struct Context {
    promise prom;
};

void callback(void* data, const char* responce, bool ok) {

}

}

task async_run(string const& request) {
    promise prom;
    auto future = prom.get_return_object();
    run(request.c_str(), detail::callback, new Context{prom});
    return future;
}
```
Чуть более реалистичная версия
```cpp
task async_run(string const& request) {
    promise prom;
    auto future = prom.get_return_object();
    auto ctx = new Context{prom};
    auto status = run(request.c_str(), detail::callback, ctx);
    if (status == MY_BAD_STATUS) {
        ctx->prom.set_error(std::make_exception_ptr(std::runtime_error("bad status")));
        delete ctx;
    }
    return future;
}
```
### А мне и на коллбеках хорошо!

// TODO: "чейнинг коллбеков"

## Под капотом
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

## Польза!

### Callback hell
Нет не такой
```cpp
do()
    .Then([]{
        return nextStep();
    })
    .Then([]{
        return lastStep();
    });
```
А примерно такой:
```cpp
void connect(T sock, T proxy) {
    auto doConnect = [=]{
        forward(sock, proxy);
        forward(proxy, sock, true);
    };
    if (proxy->state() != Connected) {
        WaitSignal(proxy, &QWebSocket::connected, sock, 5s)
            .Then(fut::Sync, doConnect)
            .Catch(fut::Sync, [](auto& e){
                log("Achtung: {}!", e.what());
            });
    } else {
        doConnect();
    }
}
```
```cpp
return WaitSignal(ws, &QWebSocket::connected, ws, 5000).ThenSync([=](auto ok){
    if (!ok) {
        logErr("Could not create proxy connection for: {}", ws->objectName());
        ws->deleteLater();
    }
    ok.get();
    return fut::Resolved();
});
```

## Undefined Behaviour (наше любимое)

### Простые случаи
```cpp
// assuming that task is some coroutine task type
task<void> f() {
    // not a coroutine, undefined behavior
}
 
task<void> g() {
    co_return;  // OK
}
 
task<void> h() {
    co_await g();
    // OK, implicit co_return;
}
```

### Реалистично опасные
```cpp
// Bad!!!
task coro(const std::vector<int>& data)
{
    co_await sleep(10s);
    // data is dangling here
    for(const auto& value : data)
        std::cout << value << std::endl;
}
```

```cpp
// OK!
task coro(const std::vector<int>& data)
{
    co_await sleep(10s);
    // data is dangling here
    for(const auto& value : data)
        std::cout << value << std::endl;
}
```
// TODO?

## Lifetime issues

Но подобные проблемы легко найти и исправить. Их последствия проявляются практически сразу и очевидно.
Настоящие проблемы с корутинами могу возникнуть в изза слабой связанности вызываемого асинхронного кода и 
вызывающей стороны (что конечно и хорошо). 

```cpp

task<void> some_func();

struct Action {
    Action(string data);
    task<void> run() {
        // ok
        co_await some_func();
        // is Action alive here? Who can tell...
    }
};

```

Поэтому нашему классу `task<T>` понадобится некий способ остановить исполнения и не вызываеть `handle.resume()`.
И это мы еще не синхронизируем взаимодействие между потоков. Корутины чисто в одном потоке уже удобны

## Execution context

// TODO

### Их можно объединить!


// TODO

## Отменяемость

// TODO

## promise.await_transform

Пример применения await_transform
```cpp
folly::coro::Task<int> task42Slow() {
  // This doesn't suspend the coroutine, just extracts the Executor*
  folly::Executor* startExecutor = co_await folly::coro::co_current_executor;
  co_await folly::futures::sleep(std::chrono::seconds{1});
  folly::Executor* resumeExecutor = co_await folly::coro::co_current_executor;
  CHECK_EQ(startExecutor, resumeExecutor);
}
```

## Компиляция

Глобальные настройи для вкелючения корутин
```cmake
if ("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
    set(CMAKE_CXX_STANDARD 20)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fcoroutines-ts")
elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
    set(CMAKE_CXX_STANDARD 20)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fcoroutines")
elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
    set(CMAKE_CXX_STANDARD 20)
endif()
```

### Боль на астре
// TODO