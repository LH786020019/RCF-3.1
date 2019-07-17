<!--
 * @Author: haoluo
 * @Date: 2019-07-16 10:38:20
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-17 18:33:38
 * @Description: file content
 -->
## 附录 —— 常见问题解答
### 1. 构建
1. 为什么我的程序试图链接到 `Boost` 库？
如果您已经定义了 `RCF_FEATURE_BOOST_SERIALIZATION`，那么 RCF 将需要链接到 `Boost.Serialization` 库。
RCF 不链接到任何其他 `Boost` 库。
`Boost` 库具有自动链接功能，可能会导致链接器查找要链接的错误文件。您可以定义 `BOOST_ALL_NO_LIB`，然后显式地告诉链接器要链接到哪些文件。
2. 我可以把 RCF 编译成 DLL 吗？
可以。要将 RCF 函数导出为 `DLL(Windows)` 或 `共享库(Unix)`，需要在构建 DLL 或共享库时定义 `RCF_BUILD_DLL`。
3. 为什么我有时会得到编译器错误当我在 RCF 头文件前 include `<windows.h>` 时？
从历史上看，Windows 平台头文件 `<windows.h>` 和  `<winsock2.h>` 存在一些头文件排序问题。包含 `<windows.h>`，在默认情况下，将包含一个较老版本的 `Winsock`，因此不可能随后在同一个翻译单元包含 `<winsock2.h>`。
这个问题最简单的解决方法是在包含 `<windows.h>` 之前定义 `WIN32_LEAN_AND_MEAN`。
4. RCF 是否在 Visual C++ 的第 4 级的编译警告是自由的？
是的，如果以下警告被禁用：
    ```output
    C4510 'class' : default constructor could not be generated
    C4511 'class' : copy constructor could not be generated
    C4512 'class' : assignment operator could not be generated
    C4127 conditional expression is constant
    ```
5. 我可以在 XYZ 平台上运行 RCF 吗？
很可能，但是您可能需要自己对 RCF 做一些小的修改，以适应平台特定的问题，比如要包含哪些平台头文件。
6. 为什么会出现链接器错误？
在 Windows 上，如果使用 TCP 传输，则需要链接到 `ws2_32.lib`。在 Unix 平台上，需要链接到 `libnsl` 或 `libsocket` 等库。
7. RCF 支持 64 位编译器吗？
支持。
8. 在 Windows 上，RCF 是否支持 Unicode 构建？
支持。
9. 为什么我的外部序列化函数会出现重复的符号链接器错误？
如果在一个头文件中定义外部序列化函数，而不使用 `inline` 修饰符，并将它们包含在两个或多个源文件中，则会得到关于重复符号(duplicate symbols)的链接器错误。解决方案是添加一个 `inline` 修饰符：
    ```cpp
    // X.hpp
    inline void serialize(SF::Archive & ar, X & x){
        ...
    }
    ```
    ，或在头文件中声明序列化函数并在源文件中定义它：
    ```cpp
    // X.hpp
    void serialize(SF::Archive & ar, X & x);
    ```
    ```cpp
    // X.cpp
    void serialize(SF::Archive & ar, X & x){
        ...
    }
    ```
10. **如何减少应用程序的构建时间？**
如果您将 RCF 头包含到您自己常用的应用程序头文件中，您可能会注意到构建时间的增加，因为编译器将为包含 RCF 头文件的每个源文件解析一次 RCF 头文件。
您应该只在需要时包含 RCF 头文件 —— 换句话说，只将它们包含到使用 RCF 函数的源文件中。在应用程序头文件中，应该能够使用前置声明，而不是包含相应的头文件。
例如，如果您定义了一个类 `X`，它有一个 `RcfClient<>` 成员，您可以前置声明 `RcfClient<>`，然后使用一个指针为成员：
    ```cpp
    // X.h
    template<typename T>
    class RcfClient;
    class SomeInterface;
    typedef RcfClient<SomeInterface> MyRcfClient;
    typedef std::shared_ptr<MyRcfClient> MyRcfClientPtr;
    class RcfServer;
    typedef std::shared_ptr<RcfServer> RcfServerPtr;
    // 包含一个 RcfClient 和一个 RcfServer 的应用程序特定的类。
    class X{  
        X();
        MyRcfClientPtr mClientPtr;
        RcfServerPtr mServerPtr;
    };

    // X.cpp
    #include "X.h"
    #include <RCF/RcfClient.hpp>
    #include <RCF/RcfServer.hpp>
    X::X() : 
        mClientPtr( new RcfClient<SomeInterface>(...) ), 
        mRcfServerPtr( new RcfServer(...) ) {}
    ```
    ```cpp
    // X.h
    template<typename T>
    class RcfClient;
    class SomeInterface;
    typedef RcfClient<SomeInterface> MyRcfClient;
    typedef std::shared_ptr<MyRcfClient> MyRcfClientPtr;
    class RcfServer;
    typedef std::shared_ptr<RcfServer> RcfServerPtr;
    // 包含一个 RcfClient 和一个 RcfServer 的应用程序特定的类。
    class X{  
        X();
        MyRcfClientPtr mClientPtr;
        RcfServerPtr mServerPtr;
    };

    // X.cpp
    #include "X.h"
    #include <RCF/RcfClient.hpp>
    #include <RCF/RcfServer.hpp>
    X::X() : 
        mClientPtr( new RcfClient<SomeInterface>(...) ), 
        mRcfServerPtr( new RcfServer(...) ) {}
    ```
    然后，您可以在应用程序的任何地方包含 `X.h`，而不需要包含任何 RCF 头文件。
### 2. 平台
1. 为什么我耗尽了 Windows XP 上的套接字句柄？
无论何时建立一个传出 TCP 连接，都必须为连接分配一个本地端口号。在 Windows XP 中，这些本地端口(有时称为临时端口)默认是从大约 4000 个端口号的范围分配的。
如果您使用 TCP 端点创建许多 `RcfClient<>` 对象，您最终将耗尽可用端口，因为在连接关闭后，Windows 会占用它们一小段时间。
您应该尽可能少地使用 TCP 连接(重用 `RcfClient<>` 对象，而不是创建新的对象)。Windows XP 上还有一些注册表设置可以缓解这个问题。找到以下键:
`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters`
，并设置
`TcpNumConnections = 0x800, MaxUserPort = 65534`
重新启动后，系统将允许扩展临时端口的范围。
这个问题只与旧版本的 Windows 有关。Windows Vista 和后面的版本使用一个扩大的临时端口范围。
２. 为什么我的 leak detector 认为 RCF 有一个 leak？
可能是因为还没有执行 [RCF::deinit()](http://www.deltavsoft.com/doc/_init_deinit_8hpp.html#af9e8b77e0223da32473bdc9be52eab56)。
3. RCF 的 TCP server 实现是否基于 I/O 完成端口？
在 Windows 上，RCF 使用 I/O 完成端口。请参阅[可伸缩性](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/performance.md)。
4. RCF 是否支持共享内存传输？
对于 Windows 上的本地 RPC, RCF 支持由共享内存支持的命名管道传输([RCF::Win32NamedPipeEndpoint](http://www.deltavsoft.com/doc/class_r_c_f_1_1_win32_named_pipe_endpoint.html))。
对于 Unix 上的本地 RPC, RCF 支持 Unix 本地套接字传输([RCF::UnixLocalEndpoint](http://www.deltavsoft.com/doc/class_r_c_f_1_1_unix_local_endpoint.html))。
5. RCF 支持 IPv6 吗？
是的，请参阅 [IPv4/IPv6](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/transports.md)。
### 3. 编程
1. 当远程调用正在进行时，如何防止用户界面失去响应？
要么在非 UI 线程上运行远程调用，要么使用进度回调函数以短时间间隔重新绘制 UI。请参阅 [Client 进度通知](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/client-side_programming.md)。
2. 如何取消长时间运行的 client 调用？
使用一个进度回调函数（ 请参阅 [Client 进度通知](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/client-side_programming.md) ）。您可以将回调函数配置为在任何给定频率调用，当您想取消调用时，抛出异常。
3. 如何在远程调用中停止 server？
您不能在远程调用中调用 [RCF::RcfServer::stop()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#acda4490f988f83af77528e3b5b5074f5)，因为 `stop()` 调用将等待所有工作线程退出，包括调用 `stop()` 的线程，从而导致死锁。
如果你真的需要在远程调用中停止 server，你可以启动一个新的线程：
    ```cpp
    RCF_BEGIN(I_Echo, "I_Echo")
        RCF_METHOD_R1(std::string, Echo, const std::string &)
    RCF_END(I_Echo)
    class EchoImpl{
    public:
        std::string Echo(const std::string &s){
            if (s == "stop") {
                // 生成一个临时线程来停止 server 。
                RCF::RcfServer * pServer = & RCF::getCurrentRcfSession().getRcfServer();
                RCF::Thread t( [=]() { pServer->stop(); } );
                t.detach();
            }
            return s;
        }
    };
    ```
4. 如何使远程访问的函数成为私有的？
让 `RcfClient<>` 成为你的实现类的friend：
    ```cpp
        RCF_BEGIN(I_Echo, "I_Echo")
            RCF_METHOD_R1(std::string, Echo, const std::string &)
        RCF_END(I_Echo)
        class EchoImpl{
        private:
            friend RcfClient<I_Echo>;
            std::string Echo(const std::string &s){
                return s;
            }
        };
    ```
5. 我如何使用单个 TCP 连接几个 `RcfClient<>` 实例？
您可以将网络连接从一个 `RcfClient<>` 移动到另一个 `RcfClient<>`。请参阅[传输访问](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/client-side_programming.md)。
6. 如何强制断开 client 与 server 的连接？
在 server 实现中调用 `RCF::getCurrentRcfSession().disconnect()`。
7. `RcfClient<>` 对象何时连接到 server ？
`RcfClient<>` 只会在您启动远程调用后建立网络连接。如果您想建立一个网络连接而不需要实际进行远程调用，那么使用 `RCF::ClientStub::connect()`。
8. 如何确定 client 从哪个 IP 地址连接？
在 server 实现中调用 `RCF::getCurrentRcfSession().getClientAddress()`。
9. 如何从 server 端代码检测 client 断开连接?？
当 client 断开连接时，server 上关联的 `RCF::RcfSession` 将被销毁。可以使用 `RcfSession::setOnDestroyCallback()`，在发生这种情况时通知应用程序代码。
    ```cpp
    void onClientDisconnect(RCF::RcfSession & session){
        // ...
    }
    class ServerImpl{
        void SomeFunc(){
            // 在远程调用中，注册一个回调函数，以便在销毁此 RcfSession 时调用。
            auto onDestroy = [&](RCF::RcfSession& session) { onClientDisconnect(session); };
            RCF::getCurrentRcfSession().setOnDestroyCallback(onDestroy);
        }
    };
    ```
10. 我可以通过防火墙使用发布/订阅吗？
是的。只要订阅者能够发起到发布者的连接，它就会接收已发布的消息。发布者将永远不会启动任何返回到发布者的网络连接。
11. 如何知道订阅者是否收到已发布的消息？
发布者不能知道这一点，因为消息是使用单向语义发布的。
12. 订阅者如何知道发布者是否已停止发布？
使用订阅者断开连接通知。请参阅[订阅者](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/publish_subscribe.md)。
还可以使用 `RCF::Subscription::isConnected()` 轮询连接性。
13. 为什么我不能在 RCF 接口中使用指针？
指针不能用作 RCF 接口中的返回类型，因为没有安全的方法来封装它们。
指针可以用作 RCF 接口中的参数，尽管大多数情况下，您最好使用引用，或者 C++ 中可用的一种智能指针类型。
14. 如何在机器上的第一个可用端口上启动 TCP server？
在传递给 `RCF::RcfServer` 构造函数的 `RCF::TcpEndpoint` 对象中指定端口号为零。启动 server 后，通过调用 `RcfServer::getIpServerTransport().getPort()` 检索端口号。
15. 如何使用相同的 RCF 接口公开多个服务对象？
使用服务绑定名称。参见[添加服务绑定](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/server-side_programming.md)。
16. 为什么我的 server 只能从本地机器访问，而不能通过网络访问？
在传递给 `RCF::RcfServer` 的 `RCF::TcpEndpoint` 中，您需要指定 `0.0.0.0` 作为 IP 地址，以允许 client 通过任何网络接口访问它。默认情况下使用的是 `127.0.0.1`，它只允许本地机器上的 client 连接。
17. 为什么我的程序在退出时崩溃或断言？
可能是因为您的程序有一个全局静态对象，其析构函数试图在 RCF 被取消初始化之后销毁 `RcfClient<>` 或 `RcfServer` 对象(或其他一些 RCF 对象)。
确保取消反初始化 RCF 之前销毁所有 RCF 对象。
18. 如何序列化 `enum`？
RCF 将自动序列化和反序列化 `C++03 enum`，作为整数表示。
`C++11 enum` 类需要一个 helper 宏 `SF_SERIALIZE_ENUM_CLASS()`（ 请参阅[标准 C++ 类型](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/serialization.md) ）。
19. 我可以使用 SF 来序列化一个对象到文件中和将文件反序列化为一个对象呢？
是的，请参阅[从磁盘和到磁盘的序列化](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/serialization.md)。
20. 如何使用 RCF 发送文件？
请参阅[文件传输](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/file_transfers.md)。
21. 如何访问 RCF server 使用的内部 `asio::io_service`？
您可以调用 `AsioServerTransport::getIoService()`：
    ```cpp
    #include <RCF/AsioServerTransport.hpp>
    RCF::RcfServer server( RCF::TcpEndpoint(0) );
    RCF::ServerTransport & transport = server.getServerTransport();
    RCF::AsioServerTransport & asioTransport = dynamic_cast<RCF::AsioServerTransport &>(transport);
    boost::asio::io_service & ioService = asioTransport.getIoService();
    ```
    当 `RcfServer` 停止时，`io_service` 将被销毁。
22. 如何在不更改 RCF 接口的情况下将安全令牌(token)从 client 传递到 server ？
您可以使用自定义请求用户数据(`RCF::ClientStub::setRequestUserData()`，`RCF::RcfSession::getRequestUserData()`)，将应用程序特定的数据从 client 传递到 server。参见[自定义请求和响应数据](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/client-side_programming.md)。
23. 我可以在 Linux 和 Windows 之间发送 `std::wstring` 对象吗？
可以。`wstring` 对象假设在 Linux 上用 `UTF-32` 表示，在 `Windows` 上用 `UTF-16` 表示，RCF 在序列化时将它们编码为 `UTF-8`。
24. 我可以在 Linux 和 Windows 之间发送 `UTF-8` 编码的 `std::string` 对象吗？
可以。RCF 将 `std::string` 序列化为一个由 8 位字符组成的序列，所以编码是 `ASCII`、`ISO-8859-1`、`UTF-8` 还是其他任何东西都无关紧要。
### 4. 其他问题
为什么文档中的许多示例中都有双括号？
以下代码片段将导致编译器错误：
```cpp
        int port = 0;
        RcfClient<I_Echo> client( RCF::TcpEndpoint(port) );
        RCF::RcfServer server( RCF::TcpEndpoint(port) );
        // 将在上面这两行上得到编译器错误…
        client.getClientStub();
        server.start();
```
由于 C++ 语言的特殊性，client 和 server 的声明实际上被解释为函数声明，使用一个名为 `port` 的 `RCF::TcpEndpoint` 参数。C++ 编译器以这种方式解释声明，以保持与 C 的向后兼容性。

为了消除这种歧义，需要在 `RCF::TcpEndpoint` 周围加上括号：
```cpp
        int port = 0;
        RcfClient<I_Echo> client(( RCF::TcpEndpoint(port) ));
        RCF::RcfServer server(( RCF::TcpEndpoint(port) ));
        // Now its OK.
        client.getClientStub();
        server.start();
```
C++ 语言的这种怪癖有时被称为[ “C++ 最烦人的解析”](http://en.wikipedia.org/wiki/Most_vexing_parse)。