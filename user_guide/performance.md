<!--
 * @Author: haoluo
 * @Date: 2019-07-16 10:25:41
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-16 10:26:54
 * @Description: file content
 -->
## 性能
远程调用
RCF的设计考虑到了两个关键的性能特性。当对单个连接执行重复的远程调用时，RCF的服务器和客户端实现中的关键代码路径遵循以下两条原则:

零复制——没有对远程调用参数或缓冲区进行内部复制。
零分配——不进行内存分配。
零拷贝
在服务器或客户机上发送或接收远程调用参数或数据时，RCF不进行任何内部复制。

但是，您应该注意，序列化可能会强制执行副本。例如，如果不复制字符串内容，反序列化std::string是不可能的，因为std::string总是分配它自己的存储。同样适用于std::vector<>。

为了通过远程调用传递数据缓冲区，而不进行任何复制，RCF提供了RCF::ByteBuffer类。ByteBuffer的内容不会在序列化或反序列化时复制，相反，RCF使用分散/聚集样式语义直接发送和接收内容。

这意味着，如果要通过远程调用传输大块的非类型化数据，应该使用RCF::ByteBuffer，而不是像std::string或std::vector这样的容器。在典型的硬件上，在一个调用中传输多个兆字节的数据不会给系统带来任何压力。

零配置
RCF在客户机和服务器代码中对关键路径进行了最小的内存分配。特别是，如果使用相同的参数进行两次远程调用，在相同的连接上，RCF不会对第二次调用进行任何堆分配，无论是在客户机上还是在服务器上。

您可以通过运行下面的代码示例来验证这一点，它覆盖了new操作符来捕获任何内存分配：
```cpp
bool gExpectAllocations = true;
// Override global operator new so we can intercept heap allocations.
void *operator new(size_t bytes)
{
    if (!gExpectAllocations)
    {
        throw std::runtime_error("Unexpected heap allocation!");
    }
    return malloc(bytes);
}
void operator delete (void *pv) throw()
{
    free(pv);
}
// Override global operator new[] so we can intercept heap allocations.
void *operator new [](size_t bytes)
{
    if (!gExpectAllocations)
    {
        throw std::runtime_error("Unexpected heap allocation!");
    }
    return malloc(bytes);
}
void operator delete [](void *pv) throw()
{
    free(pv);
}
```
```cpp
RCF_BEGIN(I_Echo, "I_Echo")
    RCF_METHOD_R1(RCF::ByteBuffer, Echo, RCF::ByteBuffer)
RCF_END(I_Echo)
class EchoImpl
{
public:
    RCF::ByteBuffer Echo(RCF::ByteBuffer byteBuffer)
    {
        return byteBuffer;
    }
};
```
```cpp
    RcfClient<I_Echo> client(( RCF::TcpEndpoint(port)));
    // First call will trigger some heap allocations.
    gExpectAllocations = true;
    client.Echo(byteBuffer);
    // These calls won't trigger any client-side or server-side heap allocations.
    gExpectAllocations = false;
    for (std::size_t i=0; i<10; ++i)
    {
        RCF::ByteBuffer byteBuffer2 = client.Echo(byteBuffer);
    }
```
在这个代码示例中，我们使用RCF::ByteBuffer作为参数，它避免了作为反序列化的一部分进行任何分配。

通常，您的代码将反序列化比RCF::ByteBuffer更复杂的对象，这些对象的反序列化可能会导致内存分配。为了消除这种分配，RCF提供了一个对象缓存，可以用来缓存常见的对象类型。

对象缓存
远程调用参数的序列化和反序列化可能成为性能瓶颈。特别是，复杂数据类型的反序列化不仅涉及首先创建对象，还涉及在反序列化对象的所有字段和子字段时的大量内存分配和CPU周期。

为了提高这些情况下的性能，RCF提供了远程调用期间使用的对象的全局缓存。在一个远程调用中用作参数的对象，可以在后续调用中透明地重用。这意味着在后续调用中可以消除由于反序列化而导致的构造开销和内存分配。

下面是一个缓存std::string对象的例子：
```cpp
RCF_BEGIN(I_Echo, "I_Echo")
    RCF_METHOD_R1(std::string, Echo, std::string)
RCF_END(I_Echo)
class EchoImpl
{
public:
    std::string Echo(const std::string & s)
    {
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
在本例中，对Echo()的第一个调用将导致多个服务器端反序列化相关的内存分配——一个用于构造std::string，另一个用于扩展字符串的内部缓冲区，以适应传入的数据。

启用对象缓存后，在调用返回后，服务器端字符串将被清除，然后保存在对象缓存中，而不是被销毁。在下一个调用中，RCF没有构造一个新的std::string，而是重用缓存中的std::string。在反序列化之后，调用std::string::resize()来适应传入的数据。由于这个特定的string对象已经在前面保存了请求的数据量，所以resize()请求不会导致任何内存分配。

对象缓存是根据每个类型配置的，使用RCF::ObjectPool::enableCaching<>()和RCF::ObjectPool::disableCaching<>()函数。对于每种缓存的数据类型，您可以指定要缓存的对象的最大数量，以及要调用哪个函数来将对象置于可重用状态。

对象缓存可以用于任何c++类型，而不仅仅是出现在RCF接口中的类型。如果服务器端代码重复创建和销毁特定类型的对象，则可以为该类型启用对象缓存。

可伸缩性
构建RCF是为了在底层硬件和操作系统允许的范围内进行扩展。

可伸缩性通常在服务器端比在客户端更受关注，因为服务器管理的网络连接往往比任何单个客户机管理的网络连接多得多。

RCF的服务器传输实现是基于Asio的，Asio是一个著名的c++网络库，多年来一直是Boost库的一部分，并且很可能成为将来c++标准网络库的基础。Asio是一个成熟且高性能的网络后端，它利用了本地网络API的可用性(Windows上的I/O完成端口、Linux上的epoll()、Solaris上的/dev/poll、FreeBSD上的kqueue())，以及没有本地网络API的性能较差的API (BSD套接字)。

因此，RCF服务器能够支持的客户机数量本质上取决于服务器可用的内核数量和内存数量，以及每个客户机所需的特定于应用程序的资源。

RCF被设计为产生最小的性能开销，考虑到网络密集、高吞吐量的应用程序。请记住，分布式系统中的瓶颈往往是由分布式系统的总体设计决定的——设计糟糕的分布式系统在通信层达到极限之前，其性能就会被大大削弱。