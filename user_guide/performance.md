<!--
 * @Author: haoluo
 * @Date: 2019-07-16 10:25:41
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-17 16:05:37
 * @Description: file content
 -->
## 性能
### 1. 远程调用
RCF 的设计考虑到了两个关键的性能特性。当对单个连接执行重复的远程调用时，RCF 的 server 和 client 实现中的关键代码路径遵循以下两条原则:
- 零复制(Zero copy) —— 没有对远程调用参数或缓冲区进行内部复制。
- 零分配(Zero allocation) —— 没有进行内存分配。
#### 1.1 零复制
在 server 或 client 上发送或接收远程调用参数或数据时，RCF 不进行任何内部复制。

但是，您应该注意，序列化可能会强制执行复制。例如，如果不复制字符串内容，反序列化 `std::string` 是不可能的，因为 `std::string` 总是分配它自己的存储。同样适用于 `std::vector<>`。

为了通过远程调用传递数据缓冲区，而不进行任何复制，RCF 提供了 [RCF::ByteBuffer](http://www.deltavsoft.com/doc/class_r_c_f_1_1_byte_buffer.html) 类。[RCF::ByteBuffer](http://www.deltavsoft.com/doc/class_r_c_f_1_1_byte_buffer.html) 的内容不会在序列化或反序列化时复制，相反，RCF 使用`分散/聚集(scatter/gather)`样式语义直接发送和接收内容。

这意味着，如果要通过远程调用传输大块的非类型化数据，应该使用 [RCF::ByteBuffer](http://www.deltavsoft.com/doc/class_r_c_f_1_1_byte_buffer.html)，而不是使用像 `std::string` 或 `std::vector` 这样的容器。在典型的硬件上，在一次调用中传输多个兆字节的数据不会给系统带来任何压力。

#### 1.2 零分配
RCF 在 client 和 server 代码中对关键路径进行了最小的内存分配。特别是，如果使用相同的参数进行两次远程调用，在相同的连接上，RCF 不会对第二次调用进行任何堆分配，无论是在 client 上还是在 server 上。

您可以通过运行下面的代码示例来验证这一点，它覆盖了 `new` 操作符来捕获任何内存分配：
```cpp
bool gExpectAllocations = true;
// 覆盖全局操作符 new，以便我们可以拦截堆分配。
void *operator new(size_t bytes){
    if (!gExpectAllocations) {
        throw std::runtime_error("Unexpected heap allocation!");
    }
    return malloc(bytes);
}
void operator delete (void *pv) throw(){
    free(pv);
}
// 覆盖全局操作符 new[]，以便我们可以拦截堆分配。
void *operator new [](size_t bytes){
    if (!gExpectAllocations) {
        throw std::runtime_error("Unexpected heap allocation!");
    }
    return malloc(bytes);
}
void operator delete [](void *pv) throw(){
    free(pv);
}
```
```cpp
RCF_BEGIN(I_Echo, "I_Echo")
    RCF_METHOD_R1(RCF::ByteBuffer, Echo, RCF::ByteBuffer)
RCF_END(I_Echo)
class EchoImpl{
public:
    RCF::ByteBuffer Echo(RCF::ByteBuffer byteBuffer){
        return byteBuffer;
    }
};
```
```cpp
    RcfClient<I_Echo> client(( RCF::TcpEndpoint(port)));
    // 第一次调用将触发一些堆分配。
    gExpectAllocations = true;
    client.Echo(byteBuffer);
    // 这些调用不会触发任何 client 端或 server 端堆分配。
    gExpectAllocations = false;
    for (std::size_t i=0; i<10; ++i) {
        RCF::ByteBuffer byteBuffer2 = client.Echo(byteBuffer);
    }
```
在这个代码示例中，我们使用 [RCF::ByteBuffer](http://www.deltavsoft.com/doc/class_r_c_f_1_1_byte_buffer.html) 作为参数，它避免了作为反序列化的一部分进行任何分配。

通常，您的代码将反序列化比 [RCF::ByteBuffer](http://www.deltavsoft.com/doc/class_r_c_f_1_1_byte_buffer.html) 更复杂的对象，这些对象的反序列化可能会导致内存分配。为了消除这种分配，RCF 提供了一种对象缓存(object cache)，可以用来缓存常见的对象类型。

### 2. 对象缓存(Object Caching)
远程调用参数的序列化和反序列化可能成为一个性能瓶颈。特别是，复杂数据类型的反序列化不仅涉及首先创建对象，还涉及在反序列化对象的所有字段和子字段时的大量内存分配和 CPU 周期。

为了提高这些情况下的性能，RCF 提供了远程调用期间使用的对象的全局缓存。在一个远程调用中用作参数的对象，可以在后续调用中透明地重用。这意味着在后续调用中可以消除由于反序列化而导致的构造开销和内存分配。

下面是一个缓存 `std::string` 对象的例子：
```cpp
RCF_BEGIN(I_Echo, "I_Echo")
    RCF_METHOD_R1(std::string, Echo, std::string)
RCF_END(I_Echo)
class EchoImpl{
public:
    std::string Echo(const std::string & s){
        return s;
    }
};
```
```cpp
    EchoImpl echo;
    RCF::RcfServer server( RCF::TcpEndpoint(0));
    server.bind<I_Echo>(echo);
    server.start();
    int port = server.getIpServerTransport().getPort();
    
    RCF::ObjectPool & cache = RCF::getObjectPool();
    // Enable caching for std::string.
    // * Don't cache more than 10 std::string objects.
    // * Call std::string::clear() before putting a string into the cache.
    auto clearString = [](std::string * pStr) { pStr->clear(); };
    cache.enableCaching<std::string>(10, clearString);
    std::string s1 = "123456789012345678901234567890";
    std::string s2;
    RcfClient<I_Echo> client(( RCF::TcpEndpoint(port) ));
    // First call.
    s2 = client.Echo(s1);
    // Subsequent calls - no memory allocations at all, in RCF runtime, or 
    // in std::string serialization/deserialization, on client or server.
    for (std::size_t i=0; i<100; ++i)
    {
        s2 = client.Echo(s1);
    }
    // Disable caching for std::string.
    cache.disableCaching<std::string>();
```
在本例中，对 `Echo()` 的第一个调用将导致多个 server 端反序列化相关的内存分配 —— 一个用于构造 `std::string`，另一个用于扩展字符串的内部缓冲区，以适应传入的数据。

**启用对象缓存后，在调用返回后，server 端字符串将被清除，然后保存在对象缓存中，而不是被销毁**。在下一个调用中，RCF 没有构造一个新的 `std::string`，而是重用缓存中的 `std::string`。在反序列化之后，调用 `std::string::resize()` 来适应传入的数据。由于这个特定的 string 对象已经在前面保存了请求的数据量，所以 `resize()` 请求不会导致任何内存分配。

对象缓存是根据每个类型配置的，使用 [RCF::ObjectPool::enableCaching<>()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_object_pool.html#ab0729ea93ed136d2a991a8f7076c235e)  和 [RCF::ObjectPool::disableCaching<>()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_object_pool.html#a102c01a6abae103bf1afc7455bf21387) 函数。对于每种缓存的数据类型，您可以指定要缓存的对象的最大数量，以及要调用哪个函数来将对象置于可重用状态。

对象缓存可以用于任何 C++ 类型，而不仅仅是出现在一个 RCF 接口中的类型。如果 server 端代码重复创建和销毁特定类型的对象，则可以为该类型启用对象缓存。

### 3. 可伸缩性
构建 RCF 是为了在底层硬件和操作系统允许的范围内进行扩展。

可伸缩性通常在 server 端比在 client 更受关注，因为 server 管理的网络连接往往比任何单个 client 管理的网络连接多得多。

RCF 的 server 传输实现是基于 [Asio](http://think-async.com/Asio/AsioStandalone) 的，`Asio` 是一个著名的 C++ 网络库，多年来一直是  [Boost](http://www.boost.org/libs/asio) 库的一部分，并且很可能成为将来 C++ 标准网络库的基础。Asio 是一个成熟且高性能的网络后端，它利用了本地网络 API 的可用性（ Windows 上的 I/O 完成端口、Linux 上的 `epoll()`、Solaris 上的 `/dev/poll`、FreeBSD 上的 `kqueue()` ），以及没有本地网络 API 的性能较差的 API(BSD 套接字)。

因此，RCF server 能够支持的 client 数量本质上取决于 server 可用的内核数量和内存数量，以及每个 client 所需的应用程序特定的资源。

RCF 被设计为产生最小的性能开销，考虑到网络密集、高吞吐量的应用程序。请记住，分布式系统中的瓶颈往往是由分布式系统的总体设计决定的 —— 设计糟糕的分布式系统在通信层达到极限之前，其性能就会被大大削弱。