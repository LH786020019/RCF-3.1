<!--
 * @Author: haoluo
 * @Date: 2019-07-12 14:38:26
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-12 15:49:48
 * @Description: file content
 -->
@[TOC](教程)
# 教程
那么，什么是远程调用框架(Remote Call Framework, RCF)？你会用它来做什么呢？

RCF 是一个 C++ 库，允许在 C++ 进程之间进行远程调用。RCF 提供了一个干净简单的解决方案，可以让多个 C++ 进程相互通信，并且开销和复杂度都降到最低。

理解 RCF 如何工作的最简单的方法是直接进入并开始一个示例。本教程介绍了一个简单的基于 RCF 的 `client` 和 `server`，并随后使用它们来介绍 RCF 的基本特性。

### 1. Getting started
#### 1.1 Hello World
我们将从一个简单的基于 TCP 的 `client` 和 `server` 开始。我们希望编写一个 server，它公开一个 `print service`，该 service 接收来自 client 的消息并将其打印到标准输出。

下面是 `server`：

```cpp
#include <iostream>
#include <RCF/RCF.hpp>
// 定义 I_PrintService RCF 接口
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
// I_PrintService RCF 接口的 server 实现
class PrintService{
public:
    void Print(const std::string & s){
        std::cout << "I_PrintService service: " << s << std::endl;
    }
};
int main(){
    try{
        // 初始化 RCF
        RCF::RcfInit rcfInit;
        // 实例化一个 RCF server
        RCF::RcfServer server(RCF::TcpEndpoint("127.0.0.1", 50001));
        // 绑定 I_PrintService 接口
        PrintService printService;
        server.bind<I_PrintService>(printService);
        // 开启 server.
        server.start();
        std::cout << "Press Enter to exit..." << std::endl;
        std::cin.get();
    } catch ( const RCF::Exception & e ){
        std::cout << "Error: " << e.getErrorMessage() << std::endl;
    }
    return 0;
}
```

和`client`：
```cpp
#include <iostream>
#include <RCF/RCF.hpp>
// 定义 I_PrintService RCF 接口.
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
int main(){
    try{
        // 初始化 RCF
        RCF::RcfInit rcfInit;
        std::cout << "Calling the I_PrintService Print() method." << std::endl;
        
        // 实例化一个 RCF client
        RcfClient<I_PrintService> client(RCF::TcpEndpoint("127.0.0.1", 50001));
        // 连接到 server 并调用 Print() 方法
        client.Print("Hello World");
    } catch ( const RCF::Exception & e ){
        std::cout << "Error: " << e.getErrorMessage() << std::endl;
    }
    return 0;
}
```
代码应该相当容易理解。
<!-- 修改Building RCF链接 -->
要构建此代码，请将其复制到一个 C++ 源文件中，然后将其与 `RCF.cpp` 一起编译（可在 `<rcf_distro>/src/RCF/RCF.cpp` 发行版中获得），其中 `<rcf_distro>` 是您解压缩 RCF 发行版的位置)。您还需要将 `<rcf_distro>/include` 添加到您的编译器 include 目录中。有关构建的更多信息可以在 [Building RCF]() 中找到。

现在让我们启动 server：
```shell
c:\Projects\RcfSample\Debug>Server.exe
Press Enter to exit...
```
然后启动 client：
```shell
c:\Projects\RcfSample\Debug>Client.exe
Calling the I_PrintService Print() method.
c:\Projects\RcfSample\Debug>
```
在 server 窗口，你现在应该会看到：
```shell
c:\Projects\RcfSample\Debug>Server.exe
Press Enter to exit...
I_PrintService service: Hello World
```
到目前为止，一切顺利。

为了更容易地运行和调试代码，我们可以重写我们的 `print` 服务应用程序，使 client 和 server 运行在一个单一的进程：
```cpp
#include <iostream>
#include <RCF/RCF.hpp>
// 定义 I_PrintService RCF 接口
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
// I_PrintService RCF 接口的 server 实现
class PrintService{
public:
    void Print(const std::string & s){
        std::cout << "I_PrintService service: " << s << std::endl;
    }
};
int main(){
    try{
        // 初始化 RCF.
        RCF::RcfInit rcfInit;
        // 实例化一个 RCF server
        RCF::RcfServer server( RCF::TcpEndpoint("127.0.0.1", 50001) );
        // 绑定 I_PrintService 接口
        PrintService printService;
        server.bind<I_PrintService>(printService);
        
        // 开启 server
        server.start();
        std::cout << "Calling the I_PrintService Print() method." << std::endl;
        // 实例化一个 RCF client
        RcfClient<I_PrintService> client( RCF::TcpEndpoint("127.0.0.1", 50001) );
        // 连接到 server，然后调用 Print() 方法
        client.Print("Hello World");
    } catch ( const RCF::Exception & e ){
        std::cout << "Error: " << e.getErrorMessage() << std::endl;
    }
    
    return 0;
}
```
运行这个程序产生的输出：
```output
Calling the I_PrintService Print() method.
I_PrintService service: Hello World
```
此刻让我们总结一下：
- 一个 TCP server 监听本地主机(127.0.0.1)接口的 50001 端口。
- 一个 client 建立单个 TCP 连接到 127.0.0.1:50001，并以明文与 server 通信。
- 此 server 能够在本地系统资源允许的范围内扩展到尽可能多的并发 client 连接。一个典型的系统可以很容易地处理数千个 client 连接。
- 如前所述，server 是单线程的。因此，无论并发连接多少 client，都没有并行存取(concurrent access) std::cout 的风险。

本教程的其余部分将以这个示例为基础，演示 RCF 的一些基本特性。
### 2. 接口
RCF 接口直接在代码中使用 C++ 定义。您可以像定义任何其他 C++ 代码一样定义 RCF 接口，然后将这些接口直接绑定到您的 C++ client 和 server 代码。

在上面的例子中，我们定义了 `I_PrintService` 接口：
```cpp
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
```
这段代码定义了 RCF 接口 `I_PrintService`。该接口包含一个方法 `Print()`。`Print()` 方法有一个 `void` 返回类型，并接受一个 `const std::string &` 参数。

`RCF_METHOD_<xx>()` 宏用于定义远程方法。`RCF_METHOD_V1()` 定义了一个带有 `void` 返回类型和一个参数的远程方法。`RCF_METHOD_R2()` 定义了一个具有非 void 返回类型和两个参数的远程方法，以此类推。`RCF_METHOD_<xx>()` 宏是为具有 void 和非 void 返回类型的方法定义的，最多使用15个参数。

在server上，我们将 RCF 接口绑定到一个服务对象上：
```cpp
server.bind<I_PrintService>(printService);
```
Server 现在将把 `I_PrintService` 接口上传入的远程调用路由到 `printService` 对象。

让我们向 `I_PrintService` 接口添加更多的远程方法。我们将添加打印一个字符串列表的方法，并返回打印的字符数：
```cpp
// Include serialization code for std::vector<>.
#include <SF/vector.hpp>
RCF_BEGIN(I_PrintService, "I_PrintService")
    // Print a string.
    RCF_METHOD_V1(void, Print, const std::string &)                         
    // Print a list of strings. Returns number of characters printed.
    RCF_METHOD_R1(int,  Print, const std::vector<std::string> &)    
    // Print a list of strings. Returns number of characters printed.
    RCF_METHOD_V2(void, Print, const std::vector<std::string> &, int &)     
RCF_END(I_PrintService)
```
将这些方法添加到 `I_PrintService` 接口后，我们还需要在 `PrintService` 服务对象中实现它们：
```cpp
class PrintService
{
public:
    void Print(const std::string & s)
    {
        std::cout << "I_PrintService service: " << s << std::endl;
    }
    int Print(const std::vector<std::string> & v)
    {
        int howManyChars = 0;
        for (std::size_t i=0; i<v.size(); ++i)
        {
            std::cout << "I_PrintService service: " << v[i] << std::endl;
            howManyChars += (int) v[i].size();
        }
        return howManyChars;
    }
    
    void Print(const std::vector<std::string> & v, int & howManyChars)
    {
        howManyChars = 0;
        for (std::size_t i=0; i<v.size(); ++i)
        {
            std::cout << "I_PrintService service: " << v[i] << std::endl;
            howManyChars += (int) v[i].size();
        }
    }
};
```
下面展示了我们如何调用新的 `Print()` 方法：
```cpp
RcfClient<I_PrintService> client( RCF::TcpEndpoint(50001) );
std::vector<std::string> stringsToPrint;
stringsToPrint.push_back("First message");
stringsToPrint.push_back("Second message");
stringsToPrint.push_back("Third message");
// Remote call returning argument through return value.
int howManyChars = client.Print(stringsToPrint);
// Remote call returning argument through non-const reference parameter.
client.Print(stringsToPrint, howManyChars);
```

### 3. 序列化
作为执行远程调用的一部分，RCF 将远程调用的参数序列化为字节序列，并通过网络连接传输它们。在连接的另一端，RCF 将参数从字节序列反序列化为实际的 C++ 对象。

RCF 自动处理大多数标准 C++ 数据类型的序列化。您所需要做的就是为正在使用的数据类型包含相关的序列化头。有关序列化的部分提供了一个标准 C++ 数据类型及其 RCF 序列化头的表。

还可以在远程调用中使用自己的应用程序数据类型作为参数。在这种情况下，需要为这些数据类型提供序列化函数。
 
到目前为止，我们只在 `I_PrintService` 接口中使用了标准的 C++ 数据类型。想象一下，现在我们想要向 `I_PrintService` 接口添加一个 `Print()` 方法，它接受一个特定于应用程序的 `LogMessage` 结构，`LogMessage` 定义如下：
```cpp
enum LogSeverity{
    Critical, Error, Warning, Informative
};

class LogMessage{
public:
    std::string     mUserName;
    int             mThreadId = 0;
    LogSeverity     mSeverity = Informative;
    int             mDurationMs = 0;
    std::string     mMessage;
};
```
在 RCF 接口中使用 `LogMessage` 类之前，我们需要为它提供一个 RCF 序列化函数。

序列化函数通常编写起来相当简单。它们可以作为成员函数提供，也可以作为与正在序列化的类相同名称空间中的独立全局函数提供。

数据类型的序列化函数本质上指定哪些成员参与序列化，并在 RCF 需要序列化或反序列化该类型的对象时调用。

序列化函数可以是成员函数，也可以是独立函数。下面是 LogMessage 的序列化成员函数：
```cpp
void LogMessage::serialize(SF::Archive& ar){
    ar & mUserName;
    ar & mThreadId;
    ar & mSeverity;
    ar & mDurationMs;
    ar & mMessage;
}
```
如果我们想使用一个独立函数，它应该是这样的：
```cpp
void serialize(SF::Archive& ar, LogMessage& msg){
    ar & msg.mUserName;
    ar & msg.mThreadId;
    ar & msg.mSeverity;
    ar & msg.mDurationMs;
    ar & msg.mMessage;
}
```
就是这样。我们现在可以向`I_PrintService`接口添加一个方法：
```cpp
RCF_BEGIN(I_PrintService, "I_PrintService")
    // Print a string.
    RCF_METHOD_V1(void, Print, const std::string &)
    // Print a LogMessage.
    RCF_METHOD_V1(void, Print, const LogMessage &)
RCF_END(I_PrintService)
```
，然后从一个 client 调用它：
```cpp
RcfClient<I_PrintService> client(RCF::TcpEndpoint(50001));
LogMessage msg;
client.Print(msg);
```
### 4. Client Stubs
远程调用总是通过RcfClient<>实例进行的。每个RcfClient<>实例都包含一个RCF::ClientStub，可以通过调用RcfClient<>::getClientStub()访问它。client stub是几乎所有远程调用的客户端配置都发生的地方。

两个重要的客户端设置是连接超时和远程调用超时。连接超时确定客户机在尝试建立到服务器的网络连接时将等待多长时间，而远程调用超时确定客户机将等待多长时间，以便从服务器返回远程调用响应。

要更改这些设置，请调用RCF::ClientStub::setConnectTimeoutMs()和RCF::ClientStub::setRemoteCallTimeoutMs()函数:
```cpp
RcfClient<I_PrintService> client( RCF::TcpEndpoint(50001) );
// 5 second timeout when establishing network connection.
client.getClientStub().setConnectTimeoutMs(5*1000);
// 60 second timeout when waiting for remote call response from the server.
client.getClientStub().setRemoteCallTimeoutMs(60*1000);
client.Print("Hello World");
```
另一个常用的客户端配置设置是远程调用进度回调，通过RCF::ClientStub::setRemoteCallProgressCallback()配置。这允许您监视或取消正在进行的远程调用：
```cpp
RcfClient<I_PrintService> client( RCF::TcpEndpoint(50001) );
auto progressCallback = [&](const RCF::RemoteCallProgressInfo & info, RCF::RemoteCallAction& action) { 
    // To cancel the call, set action to RCF::Rca_Cancel:
    //action = RCF::Rca_Cancel;
    action = RCF::Rca_Continue;
};
client.getClientStub().setRemoteCallProgressCallback(progressCallback,500);
// While the call is executing, the progress callback will be called every 500ms.
client.Print("Hello World");
```
在RCF::ClientStub中记录了许多其他客户端配置设置。您可以在客户端编程中阅读更多关于客户端配置的信息。
### 5. Server Sessions
在客户端，每个RcfClient<>实例控制到服务器的单个网络连接。在服务器端，RCF为每个到服务器的连接维护一个会话(RCF::RcfSession)。客户机连接的RcfSession可通过全局RCF::getCurrentRcfSession()函数提供给服务器端代码。

您可以使用RcfSession来维护特定于特定客户机连接的应用程序数据。通过调用RCF::RcfSession::createSessionObject<>()或RCF::RcfSession::getSessionObject<>()，可以将任意c++对象存储在会话中。

例如，我们将如何将PrintServiceSession对象与每个到I_PrintService接口的客户机连接关联起来，以跟踪在该连接上进行的调用的数量:
```cpp
class PrintServiceSession{
public:
    PrintServiceSession() : mCallCount(0){
        std::cout << "Created PrintServiceSession object." << std::endl;
    }

    ~PrintServiceSession(){
        std::cout << "Destroyed PrintServiceSession object." << std::endl;
    }
    std::size_t mCallCount;
};

class PrintService{
public:
    void Print(const std::string & s){
        RCF::RcfSession & session = RCF::getCurrentRcfSession();
        // Creates the session object if it doesn't already exist.
        PrintServiceSession & printSession = session.getSessionObject<PrintServiceSession>(true);
        ++printSession.mCallCount;
        std::cout << "I_PrintService service: " << s << std::endl;
        std::cout << "I_PrintService service: " << "Total calls on this connection so far: " << printSession.mCallCount << std::endl;
    }
};
```
不同的 `RcfClient<>` 实例将具有不同的到 server 的连接。这里我们从两个 `RcfClient<>` 实例中调用 `Print()` 三次：
```cpp
for (std::size_t i=0; i<2; ++i) {
    RcfClient<I_PrintService> client( RCF::TcpEndpoint(50001) );
    client.Print("Hello World");
    client.Print("Hello World");
    client.Print("Hello World");
}
```
，产生如下输出：

```
Created PrintServiceSession object.
I_PrintService service: Hello World
I_PrintService service: Total calls on this connection so far: 1
I_PrintService service: Hello World
I_PrintService service: Total calls on this connection so far: 2
I_PrintService service: Hello World
I_PrintService service: Total calls on this connection so far: 3
Destroyed PrintServiceSession object.
Created PrintServiceSession object.
I_PrintService service: Hello World
I_PrintService service: Total calls on this connection so far: 1
I_PrintService service: Hello World
I_PrintService service: Total calls on this connection so far: 2
I_PrintService service: Hello World
I_PrintService service: Total calls on this connection so far: 3
Destroyed PrintServiceSession object.
```
当客户机连接关闭时，将销毁服务器会话和任何关联的会话对象。

有关服务器会话的更多信息，请参见服务器端编程。
### 6. Transports
在RCF中，传输层处理跨网络连接的消息传输。传输层由传递给RcfServer和RcfClient<>构造函数的端点参数决定。到目前为止，我们一直使用RCF::TcpEndpoint来指定TCP传输。

默认情况下，当您指定只有端口号的RCF::TcpEndpoint时，RCF将使用127.0.0.1作为IP地址。所以下面两个片段是等价的:
```cpp
RCF::RcfServer server( RCF::TcpEndpoint(50001) );
```
```cpp
RCF::RcfServer server( RCF::TcpEndpoint("127.0.0.1", 50001) );
```
，详情如下：

```cpp
RcfClient<I_PrintService> client( RCF::TcpEndpoint(50001) );
```

```cpp
RcfClient<I_PrintService> client( RCF::TcpEndpoint("127.0.0.1", 50001) );
```
127.0.0.1是IPv4回送地址。监听127.0.0.1的服务器只对与服务器位于同一台机器上的客户机可用。您可能希望跨网络运行客户机，在这种情况下，服务器需要监听外部可见的网络地址。最简单的方法是指定0.0.0.0(用于IPv4)，或者::0(用于IPv6)，这将使服务器监听所有可用的网络接口：
```cpp
// Server listening for TCP connections on all network interfaces.
RCF::RcfServer server( RCF::TcpEndpoint("0.0.0.0", 50001) );
// Client connecting to TCP server on "printsvr.acme.com".
RcfClient<I_PrintService> client( RCF::TcpEndpoint("printsvr.acme.com", 50001) );
```
RCF 还支持许多其他端点类型。要在 UDP 上运行服务器和客户机，请使用 `RCF::UdpEndpoint`：
```cpp
// Server listening for UDP messages on all network interfaces.
RCF::RcfServer server( RCF::UdpEndpoint("0.0.0.0", 50001) );
// Client sending UDP messages to UDP server on "printsvr.acme.com".
RcfClient<I_PrintService> client( RCF::UdpEndpoint("printsvr.acme.com", 50001) );
```
使用UDP进行双向(请求/响应)消息传递通常是不实际的，因为UDP的不可靠语义意味着消息可能不会被传递，或者可能被无序地传递。UDP在单向消息传递场景中非常有用，在单向消息传递场景中，服务器应用程序逻辑对丢失或无序到达的消息具有弹性。

要在命名管道上运行服务器和客户机，请使用RCF::Win32NamedPipeEndpoint(用于Windows命名管道)或RCF::UnixLocalEndpoint(用于UNIX本地域套接字)：
```cpp
// Server listening for Win32 named pipe connections on pipe "PrintSvrPipe".
RCF::RcfServer server( RCF::Win32NamedPipeEndpoint("PrintSvrPipe") );
// Client connecting to Win32 named pipe server named "PrintSvrPipe".
RcfClient<I_PrintService> client( RCF::Win32NamedPipeEndpoint("PrintSvrPipe") );
```
如果需要通过HTTP协议对远程调用进行隧道传输，还可以使用HTTP和HTTPS端点。这样做最常见的原因是遍历网络基础设施，如HTTP代理：
```cpp
// Server-side.
PrintService printService;
RCF::RcfServer server( RCF::HttpEndpoint("0.0.0.0", 80) );
server.bind<I_PrintService>(printService);
server.start();
// Client-side.
// This client will connect to printsvr.acme.com via the HTTP proxy at proxy.acme.com:8080.
RcfClient<I_PrintService> client( RCF::HttpEndpoint("printsvr.acme.com", 80) );
client.getClientStub().setHttpProxy("web-proxy.acme.com");
client.getClientStub().setHttpProxyPort(8080);
client.Print("Hello World");
```
可以为一个RcfServer配置多个传输。例如，要配置一个接受TCP、UDP和命名管道连接的服务器：
```cpp
RCF::RcfServer server;
server.addEndpoint( RCF::TcpEndpoint("0.0.0.0", 50001) );
server.addEndpoint( RCF::UdpEndpoint("0.0.0.0", 50002) );
server.addEndpoint( RCF::Win32NamedPipeEndpoint("PrintSvrPipe") );
server.start();
```
有关transports的更多信息，请参见Transports。
### 7. 加密及认证
RCF为远程调用的加密和身份验证提供了许多选项。加密和身份验证作为传输协议提供，在传输之上分层。

RCF支持以下传输协议:
- [NTLM](http://msdn.microsoft.com/en-us/library/windows/desktop/aa378749%28v=vs.85%29.aspx)
- [Kerberos](http://msdn.microsoft.com/en-us/library/windows/desktop/aa378747%28v=vs.85%29.aspx)
- [SSL](http://en.wikipedia.org/wiki/Secure_Socket_Layer)

NTLM和Kerberos传输协议仅在Windows上受支持。SSL传输协议在所有平台上都受支持，但是对于非windows构建，需要使用OpenSSL支持构建RCF(请参阅构建RCF)。

一个 `RcfServer` 可以配置为要求其客户端使用某些传输协议：
```cpp
std::vector<RCF::TransportProtocol> protocols;
protocols.push_back(RCF::Tp_Ntlm);
protocols.push_back(RCF::Tp_Kerberos);
server.setSupportedTransportProtocols(protocols);
```
在客户端，RCF::ClientStub::setTransportProtocol()用于配置传输协议。例如，要在客户端连接上启用NTLM协议：
```cpp
RcfClient<I_PrintService> client( RCF::TcpEndpoint(50001) );
client.getClientStub().setTransportProtocol(RCF::Tp_Ntlm);
client.Print("Hello World");
```
在本例中，客户机将向服务器进行身份验证，并使用NTLM协议加密其连接，使用登录用户的隐式Windows凭据。

要提供显式凭据，可以使用RCF::ClientStub::setUserName()和RCF::ClientStub::setPassword()：
```cpp
RcfClient<I_PrintService> client( RCF::TcpEndpoint(50001) );
client.getClientStub().setTransportProtocol(RCF::Tp_Ntlm);
client.getClientStub().setUserName("SomeDomain\\Joe");
client.getClientStub().setPassword("JoesPassword");
client.Print("Hello World");
```
Kerberos协议也可以类似地配置。Kerberos协议要求客户机提供服务器的服务主体名称(SPN)。为此，可以调用RCF::ClientStub::setKerberosSpn()：
```cpp
RcfClient<I_PrintService> client( RCF::TcpEndpoint(50001) );
client.getClientStub().setTransportProtocol(RCF::Tp_Kerberos);
client.getClientStub().setKerberosSpn("SomeDomain\\ServerAccount");
client.Print("Hello World");
```
如果客户端试图在不配置服务器所需的传输协议的情况下调用Print()，他们将得到一个错误：
```cpp
try{
    RcfClient<I_PrintService> client( RCF::TcpEndpoint(50001) );
    client.Print("Hello World");
}catch(const RCF::Exception & e){
    std::cout << "Error: " << e.getErrorMessage() << std::endl;
}
```

```
Error: Server requires one of the following transport protocols to be used: NTLM, Kerberos.
```
在 `Print()` 的服务器端实现中，可以检索当前会话中使用的传输协议。如果传输协议是NTLM或Kerberos，还可以检索客户机的Windows用户名，甚至可以模拟客户机：
```cpp
class PrintService{
public:
    void Print(const std::string & s){
        RCF::RcfSession & session = RCF::getCurrentRcfSession();
        RCF::TransportProtocol protocol = session.getTransportProtocol();

        if ( protocol == RCF::Tp_Ntlm || protocol == RCF::Tp_Kerberos ){
            std::string clientUsername = session.getClientUsername();

            RCF::SspiImpersonator impersonator(session);
            // 现在在客户端的Windows凭据下运行。
            // ...

            // 在我们退出作用域(scope)时模拟(Impersonation)结束。
        }
        std::cout << s << std::endl;
    }
};
```
RCF还支持使用SSL作为传输协议。RCF提供了两种SSL实现，一种基于OpenSSL，另一种基于Windows Schannel安全包。Windows Schannel实现在Windows上自动使用，而如果在构建RCF时定义了RCF_FEATURE_OPENSSL=1，则使用OpenSSL实现(参见构建RCF)。

启用SSL的服务器需要向客户端提供SSL证书。证书配置的机制因使用的实现(Schannel或OpenSSL)而异。下面是一个使用Schannel SSL实现的例子:
```cpp
RCF::RcfServer server( RCF::TcpEndpoint(50001) );
RCF::CertificatePtr serverCertificatePtr( new RCF::PfxCertificate(
    "C:\\ServerCert.p12", 
    "Password",
    "CertificateName") );

server.setCertificate(serverCertificatePtr);
server.start();

RcfClient<I_PrintService> client( RCF::TcpEndpoint(50001) );
client.getClientStub().setTransportProtocol(RCF::Tp_Ssl);
client.getClientStub().setEnableSchannelCertificateValidation("localhost");
```
PfxCertificate用于从.pfx和.p12文件加载PKCS #12证书。RCF还提供了RCF::StoreCertificate类，用于从Windows证书存储中加载证书。

如果您使用的是OpenSSL，则可以使用RCF::PemCertificate类从. PEM文件加载PEM证书。

RCF支持使用zlib对远程调用进行传输级压缩。压缩使用ClientStub::setEnableCompression()独立于传输协议进行配置。压缩阶段紧接在传输协议阶段之前应用。换句话说，远程调用数据在加密之前被压缩。

这是一个客户端通过HTTP代理连接到服务器的例子，使用NTLM进行身份验证和加密，并启用压缩：
```cpp
RcfClient<I_PrintService> client( RCF::HttpEndpoint("printsvr.acme.com", 80) );
client.getClientStub().setHttpProxy("web-proxy.acme.com");
client.getClientStub().setHttpProxyPort(8080);
client.getClientStub().setTransportProtocol(RCF::Tp_Ntlm);
client.getClientStub().setEnableCompression(true);
client.Print("Hello World");
```
有关更多信息，请参见Transport protocols。
### 8. Server Threading
默认情况下，RcfServer使用一个线程来处理传入的远程调用，因此远程调用是一个接一个地串行分配的。即使您在服务器中配置了多个传输，这些传输都将由一个线程提供服务。

这通常是运行服务器最安全的方式，因为服务器端应用程序代码中不需要线程同步。您还应该注意，单线程服务器根本不限制可以同时访问服务器的客户机数量。它只限制可以并发执行的远程调用的数量。如果应用程序中的远程调用执行得很快，那么很可能单线程服务器就足以处理大量客户机。

然而，执行需要一些时间的远程调用也很常见。例如，您的服务器端代码可能正在向其他后端服务器发出请求，这可能会造成严重的延迟。如果后端服务器响应较慢，那么RCF服务器线程将被阻塞一段时间，其他客户机请求将被延迟。在这种情况下，您将需要一个多线程服务器，以便能够继续响应其他客户机。

要配置多线程RcfServer，可以使用RCF::RcfServer::setThreadPool()来为服务器分配一个线程池。RCF线程池可以配置为固定数量的线程：
```cpp
RCF::RcfServer server( RCF::TcpEndpoint(50001) );
// 具有固定线程数(5)的线程池。
RCF::ThreadPoolPtr threadPoolPtr( new RCF::ThreadPool(5) );
server.setThreadPool(threadPoolPtr);
server.start();
```
，或随服务器负载变化的动态线程数：
```cpp
RCF::RcfServer server( RCF::TcpEndpoint(50001) );
// 具有不同数量线程(1到25个)的线程池。
RCF::ThreadPoolPtr threadPoolPtr( new RCF::ThreadPool(1, 25) );
server.setThreadPool(threadPoolPtr);
server.start();
```
通过RCF::RcfServer::setThreadPool()配置的线程池在RcfServer内的所有传输中共享。如果希望为特定传输配置专用线程池，可以使用RCF::ServerTransport::setThreadPool()：
```cpp
RCF::RcfServer server;
RCF::ServerTransport & tcpTransport = server.addEndpoint(RCF::TcpEndpoint(50001));
RCF::ServerTransport & pipeTransport = server.addEndpoint(RCF::Win32NamedPipeEndpoint("MyPipe"));

// 最多有5个线程来服务TCP客户机的线程池。
RCF::ThreadPoolPtr tcpThreadPoolPtr( new RCF::ThreadPool(1, 5) );
tcpTransport.setThreadPool(tcpThreadPoolPtr);

// 使用单线程的线程池来服务命名管道客户端。
RCF::ThreadPoolPtr pipeThreadPoolPtr( new RCF::ThreadPool(1) );
pipeTransport.setThreadPool(pipeThreadPoolPtr);

server.start();
```
### 9. 异步远程调用
RCF通常同步执行远程调用，并将阻塞客户机线程，直到调用完成。RCF还允许异步执行远程调用。不是阻塞客户机线程，而是在稍后远程调用完成时通知客户机线程。

使用RCF::Future<>模板在RCF中执行异步远程调用。本质上，对于远程调用中的任何参数或返回类型T，如果您传递的是将来<t>，则远程调用将异步执行。</t>可以将多个Future<t>实例传入远程调用。</t>您可以配置Future<t>实例，以便在完成时接收回调，也可以使用RCF::Future::wait()或RCF::Future::ready()检查远程调用的状态。</t>远程调用完成后，可以使用RCF::Future::operator*()取消对Future<t>实例的引用，并检索实际的参数值。</t>

给定这个RCF接口：
```cpp
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_R1(int, Print, const std::string &)
RCF_END(I_PrintService)
```
，这是一个客户端进行异步调用，然后轮询结果可用的例子：
```cpp
RcfClient<I_PrintService> client( RCF::TcpEndpoint(50001) );
// Asynchronous remote call.
RCF::Future<int> fRet = client.Print("Hello World");
// Wait for the remote call to complete.
while ( !fRet.ready() ) {
    RCF::sleepMs(1000);
}
// Check for errors.
std::unique_ptr<RCF::Exception> ePtr = client.getClientStub().getAsyncException();
if ( ePtr ) {
    std::cout << "Error: " << ePtr->getErrorMessage();
} else {
    std::cout << "Characters printed: " << *fRet;
}
```
当然，轮询完成与阻塞调用并没有太大的不同。一个更强大的技术是为远程调用分配一个完成回调。完成回调函数是一个函数对象，RCF将在远程调用完成后调用该函数。下面是一个简单的例子：
```cpp
typedef std::shared_ptr< RcfClient<I_PrintService> > PrintServicePtr;
void onPrintCompleted(
    RCF::Future<int> fRet, 
    PrintServicePtr clientPtr)
{
    std::unique_ptr<RCF::Exception> ePtr = clientPtr->getClientStub().getAsyncException();
    if ( ePtr )
    {
        std::cout << "Error: " << ePtr->getErrorMessage();
    }
    else
    {
        std::cout << "Characters printed: " << *fRet;
    }
}
```
```cpp
PrintServicePtr clientPtr( new RcfClient<I_PrintService>(RCF::TcpEndpoint(50001)) );
RCF::Future<int> fRet;
// Asynchronous remote call, with completion callback.
auto onCompletion = [&]() { onPrintCompleted(fRet, clientPtr); };
fRet = clientPtr->Print(RCF::AsyncTwoway(onCompletion), "Hello World");
```
注意，我们已经将未来<int>返回值和引用计数的客户机(PrintServicePtr)传递给回调函数。</int>如果我们要销毁主线程上的客户机，异步调用将被自动取消，回调将根本不会被调用。

异步远程调用在很多情况下都很有用。下面是一个客户机在50个不同的服务器上每10秒调用Print()一次的例子：
```cpp
void onPrintCompleted(PrintServicePtr clientPtr);
void onWaitCompleted(PrintServicePtr clientPtr);
void onPrintCompleted(PrintServicePtr clientPtr){
    // Print() call completed. Wait for 10 seconds.
    auto onCompletion = [&]() { onWaitCompleted(clientPtr); };
    clientPtr->getClientStub().wait(
        onCompletion,
        10 * 1000);
}
void onWaitCompleted(PrintServicePtr clientPtr){
    // 10 second wait completed. Make another Print() call.
    auto onCompletion = [&]() { onPrintCompleted(clientPtr); };
    clientPtr->Print( 
        RCF::AsyncTwoway(onCompletion), 
        "Hello World");
}
```

```cpp
// Addresses to 50 servers.
std::vector<RCF::TcpEndpoint> servers(50);
// ...
// Create a connection to each server, and make the first call.
std::vector<PrintServicePtr> clients;
for (std::size_t i=0; i<50; ++i)
{
    PrintServicePtr clientPtr( new RcfClient<I_PrintService>(servers[i]) );
    clients.push_back(clientPtr);
    // Asynchronous remote call, with completion callback.
    auto onCompletion = [&]() { onPrintCompleted(fRet, clientPtr); };
    clientPtr->Print( 
        RCF::AsyncTwoway(onCompletion), 
        "Hello World");
}
// All 50 servers are now being called once every 10 s.
// ...
// Upon leaving scope, the clients are all automatically destroyed. Any
// remote calls in progress are automatically canceled.
```
如果必须使用同步调用来编写这段代码，则需要50个线程，每个线程连接到一个服务器。通过使用异步调用，我们可以用一个线程管理所有50个连接。对50台服务器的远程调用都在后台RCF线程上完成，当主线程超出作用域时，连接将自动销毁。

有关更多信息，请参见异步远程调用。
### 10. Publish/subscribe
假设我们的网络上有许多I_PrintService服务器，并且我们希望客户机对它们进行相同的Print()远程调用。要做到这一点，使用常规的双向远程调用将是乏味的。首先，我们必须以某种方式维护当前可用的I_PrintService服务器列表。假设我们有这样一个列表，那么我们必须手动调用每个服务器上的Print()，并使用相同的消息。

这种场景更适合于通常称为发布/订阅的消息传递范例。发布/订阅范式基于一个发布者维护多个发布主题，以及多个订阅者订阅这些主题。每当发布者发布关于某个主题的消息时，该消息将发送给订阅该特定主题的所有订阅者。

发布/订阅系统的主要优点是，发布代码不需要知道关于订阅者的任何信息。订阅者自己负责注册他们感兴趣的主题，而发布者只负责发布消息。

RCF使设置发布/订阅系统变得很容易。发布/订阅功能由RCF::RcfServer::createPublisher()和RCF::RcfServer:: createsub()函数提供。例如，下面是发布者每秒钟发布一次Print()调用的代码：
```cpp
RCF::RcfServer publishingServer( RCF::TcpEndpoint(50001) );
publishingServer.start();
// Start publishing.
typedef std::shared_ptr< RCF::Publisher<I_PrintService> > PrintServicePublisherPtr;
PrintServicePublisherPtr pubPtr = publishingServer.createPublisher<I_PrintService>();
while (shouldContinue()) {
    Sleep(1000);
    // Publish a Print() call to all currently connected subscribers.
    pubPtr->publish().Print("Hello World");
}
// Close the publisher.
pubPtr->close();
```
创建对该发布者的订阅同样简单：
```cpp
// Start a subscriber.
RCF::RcfServer subscriptionServer(( RCF::TcpEndpoint() ));
subscriptionServer.start();
PrintService printService;
RCF::SubscriptionPtr subPtr = subscriptionServer.createSubscription<I_PrintService>(
    printService, 
    RCF::TcpEndpoint(50001));
// At this point Print() will be called on the printService object once a second.
// ...
// Close the subscription.
subPtr->close();
```
发布者发出的每个调用都作为单向调用发送给所有订阅者。

有关发布/订阅消息传递的更多信息，请参见 Publish/subscribe。
### 11. 文件传输
文件下载和上传在分布式系统中很常见。对于较小的文件，您可以将它们加载到std::string(或者更好的方法是RCF::ByteBuffer)中，然后将它们发送出去。但是，对于足够大到超过连接的最大消息长度的文件，这种方法就失效了。

RCF提供了专门的下载和上传文件的功能，而不是要求您将文件分割成块并逐个发送这些块。这些函数负责必要的分块逻辑，以及并发网络和磁盘I/O的低层细节，这些细节对于最大限度地提高吞吐量是必要的。RCF还提供了恢复中断传输的能力，以及限制文件传输消耗的带宽。

例如，下面是我们如何在I_PrintService接口上实现PrintFile()方法：
```cpp
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
    RCF_METHOD_V1(void, PrintFile, const std::string &)
RCF_END(I_PrintService)
class PrintService{
public:
    void Print(const std::string & s){
        std::cout << "I_PrintService service: " << s << std::endl;
    }
    void PrintFile(const std::string & uploadId){
        std::cout << "I_PrintService service: " << uploadId << std::endl;
        RCF::RcfSession& session = RCF::getCurrentRcfSession();
        RCF::Path pathToFile = session.getUploadPath(uploadId);
        
        // Print file
        // ...
        // Delete file once we're done with it.
        namespace fs = std::experimental::filesystem;
        fs::remove(pathToFile);
    }
};
```
```cpp
PrintService printService;
RCF::RcfServer server( RCF::TcpEndpoint(50001) );
server.bind<I_PrintService>(printService);
server.setUploadDirectory("C:\\temp");
server.start();
```
注意，PrintFile()方法采用字符串形式的上传标识符。要调用PrintFile()，客户端需要首先使用RCF::ClientStub::uploadFile()，来上载相关文件，并获取用于上载的字符串标识符。然后将字符串标识符传递给PrintFile()， PrintFile()定位服务器上的上载，打印文件，然后删除它。
```cpp
// Upload files to server.
RCF::Path pathToFile = "C:\\LargeFile.log";
std::string uploadId = RCF::generateUuid();
RcfClient<I_PrintService> client( RCF::TcpEndpoint(50001) );
client.getClientStub().uploadFile(uploadId, pathToFile);
client.PrintFile(uploadId);
```
下载文件的过程与此类似。下面是我们如何实现GetPrintSummary()方法，它的目的是向客户端返回一个可能很大的文件：
```cpp
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
    RCF_METHOD_R0(std::string, GetPrintSummary)
RCF_END(I_PrintService)
class PrintService{
public:
    void Print(const std::string & s){
        std::cout << "I_PrintService service: " << s << std::endl;
    }
    std::string GetPrintSummary(){
        RCF::Path file = "path/to/download";
        std::string downloadId = RCF::getCurrentRcfSession().configureDownload(file);
        return downloadId;
    }
};
```
GetPrintSummary()在服务器上配置下载，然后将下载标识符返回给客户机。然后，客户机调用带有下载标识符的RCF::ClientStub::downloadFile()来下载文件：
```cpp
RcfClient<I_PrintService> client( RCF::TcpEndpoint(50001) );
std::string downloadId = client.GetPrintSummary();
RCF::Path downloadTo = "C:\\downloads";
client.getClientStub().downloadFile(downloadId, downloadTo);
```
有关更多信息，请参见文件传输。