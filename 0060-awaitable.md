```cpp
struct awaitable
{
    bool await_ready() { return false; }
    void await_suspend(std::coroutine_handle<> h) {}
    void await_resume() {}
};
```