# TODO: coro concept, meaning of customization points

```cpp
template< class Promise = void >
struct coroutine_handle;

template<>
struct coroutine_handle<void>;

template<>
struct coroutine_handle<std::noop_coroutine_promise>;

using noop_coroutine_handle = std::coroutine_handle<std::noop_coroutine_promise>;
```