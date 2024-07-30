# В паутине корутин

## Зачем

### К сожалению или счастью
При работе с асинхроным кодом приходится писать следующее
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
                //handle_error()
                return; //cannot throw here
            }
            self->write(read);
        });
    }
    void write(size_t amount) {
        async_write(sock, buffer(buff, amount), [self = shared_from_this()](auto ec, size_t){
            if (ec) {
                //handle_error()
                return; //cannot throw here
            }
            self->read();
        });

    }
};
```
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
## Предыстория

Мелвин Эдвард Конвей - американский ученый-компьютерщик, программист и хакер, который придумал то, что сейчас известно как закон Конвея: "Организации, разрабатывающие системы, вынуждены создавать проекты, которые являются копиями коммуникационных структур этих организаций". 

Помимо вышеперечисленного, Конвей, пожалуй, наиболее известен благодаря разработке концепции сопрограмм. Конвей ввел термин "сопрограмма" в 1958 году и был первым, кто применил эту концепцию. Позже он написал основополагающую статью на тему сопрограмм под названием "Design of a Separable Transition-diagram".

TODO: https://habrastorage.org/webt/1o/v4/-v/1ov4-vv40pxmehwic63rgzvr5qq.jpeg
https://blog.skillfactory.ru/wp-content/uploads/2023/02/coroutine1-3919182.png


## Stackless vs Stackfull


### Stackful

Стэкфул корутины очень похожи на потоки.
Альтернативные нзвания: зеленые потоки, файберы

В отличии от потоков их конкурентность называется **Кооперативной**, так как они
управляются не ядром, которое может забрать управление в любой момент (по прерыванию, **Упреждающяя** конкурентность),
а обычно управляются **Userspace** планировщиком.

Также как и полноценные *настоящие* системные потоки они требуют от языка возможности сохранять контекст исполнения. Под
стеком подразумеваются данные на стеке и регистры процессора.

Необязательно все на одном физическом потоке (M/N)

Приемущества:
* В использовании очень похои на обычные треды без спец синтаксиса
* Не требуют помощи компилятора

Недостатки:
* Дорогое переключение контекста
* Невозможность* роста стэка

Применения:
* Boost.Coroutine(2)
* Boost.Fiber
* Userver
* Golang
* Java virtual threads
* RTOS

TODO: картинки!

### Stackless

Стэклесс корутины в своем названии и описывают основное приемущество перед стэклесс.
Сохранение точки исполнения делается с помощью чуть ли не одной цифры.

Преимущества:
* Меньшее использование памяти
* Не требует сохранения контекста
* Гибкость (генераторы)

Недостатки:
* Требуют помощи от компилятора
* Специальный синтаксис
* На практике разбитые аллокации могут быть менее эффективные, чем одним блоком под весь стэк

Применения:
* Python asyncio
* C++20
* TODO: more

TODO: картинки!

## C++20

Чаще всего для индикации завершения операции используется callback функция. 
Детали не так важны, передается ли некий `void* data` или используется полноценное стирание типов (как например `std::function<>`)

И такой код может вполне быть успешным полезным. Я сам считаю что это оптимальный вариант при написании чего то низкоуровневого. Но при написании именно бизнес-логики, а не инфраструктуры иногда удобней использовать модель `async\await`. Самые болезненные случаи будут рассмотрены дальше

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

```cpp
coroutine my_function() {
    co_return 1; // co_yield / co_await
}
```
```cpp
template<typename T>
struct coroutine {
    using promise_type = promise;
    awaitable operator co_await(); // Либо сам является awaitable
}
```
```cpp
struct promise {
    coroutine get_return_object();
    awaitable initial_suspend() noexcept;
    awaitable final_suspend() noexcept;

    // these can return awaitable themselves (TODO: check)
    void return_value(T&& result) {} // Либо эта версия
    void return_void() {} // Либо эта

    void unhandled_exception() {} // вызывается внутри catch-блока.

    // Advanced:
    T await_transform(<expr>);
    static coroutine get_return_object_on_allocation_failure();
    void* operator new(std::size_t n) noexcept;
    awaitable yield_value(T&& value);
}
```
```cpp
struct awaitable {
    bool await_ready();
    bool await_suspend();
    T await_resume(); // may throw exceptions
};
```

## Создаем свою имплементацию

### Представим, что мы имеем следующее асинхронное Апи

Аля Redis
```cpp
// ok == false => result contains exception msg
void run(string request, std::function<void(bool ok, string result)> cb);
```

```cpp
void doMultiStepTask(std::function<void(bool ok)> callback) {
    run("GET key", [=](bool ok, string result){
        if (!ok) {
            handle_error(result);
            return;
        }
        run("SET another", [=](bool ok, string result){
            if (!ok) {
                // <breakpoint>
                // doMultiStepTask(std::__1::function<void, bool>)::<lambda1>::operator()(bool, std::basic_string<char>)::<lambda2>::operator()(bool, std::basic_string<char>)
                handle_error(result);
                return;
            }
        });
    });
}
```

### Силой 20-х плюсов и священного комитета сварим следующий сниппет
```cpp
// half-Slideware ahead
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

Цель - написать полностью имплементацию для:
```cpp
task async_run(string request) ;
```
Напишем наши корутины следующим образом: все состояние одной асинхронной транзакции будет общим 
между `promise` (передающая сторона) и `task` (принимающая сторона)
```cpp
struct state {
    bool done = false;
    std::exception_ptr exc;
    string result;
    std::coroutine_handle<> handle = {}; //type-erased

    ~state() {
        if (handle) handle.destroy();
    }
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
Напомню чего мы избегаем
+: явная стейт машина
-: спаггети
```cpp
void async_main();
namespace detail {

void cb(bool ok, string result);
void cb(bool ok, string result) {
    if (ok) {
        log(result);
        async_main();
    }
}

} //detail
void async_main() {
    run("ping", detail::cb);
}
```
Можно чуть более кратко
```cpp
void async_main() {
    run("ping", [](bool ok, string result) {
        if (ok) {
            log(result);
            async_main();
        }
    });
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

inline void callback(void* data, const char* responce, bool ok) {
    auto* ctx = static_cast<Context*>(data);
    if (ok) {
        ctx->prom(responce); // promise_type::set_value() should be noexcept
    } else {
        ctx->prom(std::make_exception_ptr(std::runtime_error(responce)));
    }
    delete ctx;
}
}
```
```cpp
task async_run(string const& request) {
    promise prom;
    auto future = prom.get_return_object();
    run(request.c_str(), detail::callback, new Context{prom});
    return future;
}
```
Или так
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
```
```cpp
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
Not really
```cpp
action()
    .Then([]{
        return nextStep();
    })
    .Then([]{
        return lastStep();
    });
```
### Условия
manual `awaitable.await_ready()`
```cpp
using namespace std::chrono_literals;

void connect(T sock, T proxy) {
    auto doConnect = [=]{
        forward(sock, proxy);
        forward(proxy, sock, true);
    };
    if (proxy->state() != Connected) {
        WaitSignal(proxy, &QWebSocket::connected, sock, 5s)
            .ThenSync(doConnect)
            .CatchSync([](std::exception& e){
                log("Achtung: {}!", e.what());
            });
    } else {
        doConnect();
    }
}
```
```cpp
Future<void> connect(QWebSocket* sock, QWebSocket* proxy) {
    auto doConnect = [=]{
        forward(sock, proxy);
        forward(proxy, sock, true);
    };
    if (proxy->state() != Connected) {
        co_await WaitSignal(proxy, &QWebSocket::connected, sock, 5s);
    } else {
        doConnect();
    }
}
```
```cpp
Future<void> connect(QWebSocket* sock, QWebSocket* proxy) {
    if (proxy->state() != Connected) {
        co_await WaitSignal(proxy, &QWebSocket::connected, sock, 5s);
    }
    forward(sock, proxy);
    forward(proxy, sock, true);
}
```
### try-catch
```cpp
Future<void> func(QWebSocket* ws) {
    // ...
    return WaitSignal(ws, &QWebSocket::connected, ws, 5000).ThenSync([=](auto ok){
        if (!ok) {
            logErr("Ws error in: {}", ws->objectName());
            ws->deleteLater();
        }
        ok.get();
        return fut::Resolved();
    });
}
```
```cpp
Future<void> func(QWebSocket* ws) {
    // ...
    try {
        co_await WaitSignal(ws, &QWebSocket::connected, ws, 5000);
    } catch (...) {
        logErr("Ws error in: {}", ws->objectName());
        ws->deleteLater();
        throw;
    }
}
```

### Длинная последовательность действий
```cpp
Session sess; //move-only type
Session::Impl* d = sess.d.get();
//...
//...
return WaitSignal(d->ws, &QWebSocket::connected, d->ws, 5000)
    .ThenSync([d]{
        return d->initSession();
    })
    .ThenSync([d](Session result){
        return d->handleSession(result);
    })
    .ThenSync([MV(sess)]() mutable { //forced to use mutable lambda
        // const-correctnes in callback hell is pretty difficult!
        return std::move(sess);
    });
```
```cpp
Session sess;
//...
//...
co_await WaitSignal(sess.d->ws, &QWebSocket::connected, sess.d->ws, 5000);
auto result = co_await sess.d->initSession();
co_await sess.d->handleSession(result);
co_return sess; //auto move
```

### Циклы
```cpp
Future<void> FileSystemCache::Cleanup(const cache::CleanupParams &params) try {
    Dirs dirs;
    auto iter = std::filesystem::recursive_directory_iterator(cacheDir);
    auto end = std::filesystem::recursive_directory_iterator();
    for (;iter != end; ++iter) {
        if (iter->is_directory()) {
            size_t depth = size_t(iter.depth());
            if (dirs.size() <= depth) {
                dirs.resize(depth + 1);
            }
            dirs[depth].push_back(Path(iter->path()));
        }
    }
    if (!dirs.empty()) {
        return populateCleanups(params, dirs);
    } else {
        return fut::Resolved();
    }
} catch (...) {
    return fut::Rejected<void>(std::current_exception());
}
```
```cpp
static Future<void> populateCleanups(cache::CleanupParams params, Dirs& dirs) {
    assert(!dirs.empty());
    auto level = std::move(dirs.back());
    dirs.pop_back();
    std::vector<Future<void>> levelFuts;
    for (auto& dir: level) {
        levelFuts.push_back(doCleanup(params, dir));
    }
    return Gather(std::move(levelFuts))
        .Then(fut::Sync, [MV(dirs), MV(params)]() mutable {
            if (dirs.empty()) {
                return fut::Resolved();
            } else {
                return populateCleanups(params, dirs);
            }
        });
}
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
task coro(const std::vector<int>& _data)
{
    auto data = _data;
    co_await sleep(10s);
    for(const auto& value : data)
        std::cout << value << std::endl;
}
```

## Lifetime issues

Но подобные проблемы легко найти и исправить. Их последствия проявляются практически сразу и очевидно.
Настоящие проблемы с корутинами могут возникнуть в изза слабой связанности вызываемого асинхронного кода и 
вызывающей стороны (связаны они только через `coroutine_type` и `awaitable`, который он порождает). 

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


## Отменяемость

Связано с прошлым пунктом.

```cpp
struct Worker : std::enable_shared_from_this<Worker> {
    void run() {
        longTask([self = weak_from_this()]{ //shared_from_this
            if (auto worker = self.lock()) {
                worker->onDone();
            }
        });
    }
    void onDone() {
        //...
    }
}
```

## Execution context

```cpp
void job(Callback callback) {
    std::thread thread([=]{
        auto result = longTask();
        callback(result);
    });
    thread.detach();
}
```

### Их можно объединить!

Пример для куобъектов
```cpp
class QObjExecutor final : public fut::Executor
{
    QPointer<QObject> context;
public:
    QObjExecutor(QObject* ctx) :
        ctx(context)
    {}
    void Execute(Job job) noexcept override {
        auto ctx = context.data();
        if (!ctx) {
            return;
        }
        if (QThread::currentThread() != ctx->thread()) {
            QMetaObject::invokeMethod(ctx, job, Qt::QueuedConnection);
        } else {
            job();
        }
    }
};
```

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
```cpp
//...
  template <typename Awaitable>
  auto await_transform(NothrowAwaitable<Awaitable>&& awaitable) {
    bypassExceptionThrowing_ = BypassExceptionThrowing::REQUESTED;
    return await_transform(awaitable.unwrap());
  }
  auto await_transform(folly::coro::co_current_executor_t) noexcept {
    return ready_awaitable<folly::Executor*>{executor_.get()};
  }
```


### coroutine_handle.pointer()
Универсальный способ, если мы не управляем кодом promise. Минус - оверхед от `suspend`
```cpp
struct co_get_handle
{
    std::coroutine_handle<> _handle;
    bool await_ready() const noexcept { return false; } //sleep
    bool await_suspend(std::coroutine_handle<> handle) noexcept { _handle = handle; return false; } //get handle and continue
    std::coroutine_handle<> await_resume() noexcept { return _handle; }
};
```
```cpp
inline void callback(void* data, const char* responce, bool ok) {
    auto* ctx = static_cast<Context*>(data);
    if (ok) {
        ctx->prom(responce); // promise_type::set_value() should be noexcept
    } else {
        ctx->prom(std::make_exception_ptr(std::runtime_error(responce)));
    }
    delete ctx;
}
task async_run(string const& request) {
    auto self = co_await co_get_handle();
    auto future = prom.get_return_object();
    run(request.c_str(), detail::callback, new Context{prom});
    co_return future;
}
```
### Что за pointer

```cpp
struct coroutine_frame // <-- pointer*
{
    void (*resume)(coroutine_frame *);
    promise_type promise; // <-- promise
    int16_t state;
    bool heap_allocated;
    // args
    // locals
    //...
};
```

### Suspend-less way (more verbose and intrusive)
Универсальный способ, если мы не управляем кодом promise. Минус - оверхед от `suspend`
```cpp
struct co_get_handle {};
struct give : std::suspend_never {
    handle handle;
    handle await_resume() {
        return handle;
    }
};
struct promise
{
    // ...
    using handle = std::couroutine_handle<promise>;
    // must return awaitable
    give await_transform(co_get_handle) {
        return {handle::from_promise(*this)};
    }
};
// ACHTUNG: https://stackoverflow.com/questions/76110225/why-is-promise-typeawait-transform-greedy
```
```cpp
inline void callback(void* data, const char* responce, bool ok) {
    auto self = std::coroutine_handle<task::promise_type>::from_pointer(data);
    auto& prom = self.promise();
    if (ok) {
        prom(responce);
    } else {
        prom(std::make_exception_ptr(std::runtime_error(responce)));
    }
}
task async_run(string const& request) {
    auto self = co_await co_get_handle();
    run(request.c_str(), detail::callback, self.pointer());
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

- Хочу корутины
- astra 1.7 target...
- look inside
- gcc8
- sad.png
- clang14 (TODO: check)
- ща все врубим
- libstdc++ от gcc8
- sad2.png
- dual-stdlibs in project moment


## Готовые решения

* asio!
* folly?
* qcoro!
* https://github.com/andreasbuhr/cppcoro!?