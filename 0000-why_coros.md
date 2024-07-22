# Зачем корутины/сопрограммы?

## Все мы хотим писать такой код:
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
## К сожалению или счастью приходится писать следующее:
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

[Далее](001-melvin_edward_conway.md)