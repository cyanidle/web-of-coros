
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