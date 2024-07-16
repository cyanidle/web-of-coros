```cpp
#include <string>
#include <functional>
#include <coroutine>
#include <stdexcept>
#include <optional>
#include <concepts>
#include <memory>

using std::string;

// ok == false => result contains exception msg
void run(string request, std::function<void(bool ok, string result)> cb);
void log(string msg);

struct state {
    bool done = false;
    std::exception_ptr exc;
    string result;
    std::coroutine_handle<> handle = {}; //type-erased
};

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

// Клей со старым кодом!
task async_run(string request) {
    promise prom;
    auto future = prom.get_return_object();
    run(request, [prom](auto ok, string result){
        if (ok) {
            prom.return_value(std::move(result));
        } else {
            prom.set_error(std::make_exception_ptr(std::runtime_error(std::move(result))));
        }
    });
    return future;
}


task async_main() {
    while (true) {
        auto pong = co_await async_run("ping");
        log(pong);
    }
}
```