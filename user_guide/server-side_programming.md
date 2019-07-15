<!--
 * @Author: haoluo
 * @Date: 2019-07-15 11:50:47
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-15 16:33:03
 * @Description: file content
 -->
## 服务器端编程
在服务器端，RCF::RcfServer是负责从客户端发送远程调用的基本类。RcfServer包含一个或多个传输，它侦听来自客户机的远程调用。RcfServer还公开一个或多个服务绑定，它将来自客户机的远程调用分派给这些绑定。

本教程展示了一个服务器的最小示例。首先，使用TCP传输创建RcfServer：
```cpp
RCF::RcfServer server( RCF::TcpEndpoint(50001) );
```
然后为I_PrintService接口配置一个服务绑定:
```cpp
PrintService printService;
server.bind<I_PrintService>(printService);
```
最后，服务器启动并开始响应来自客户端的远程调用：
```cpp
server.start();
```
配置一个服务器
增加传输
RcfServer侦听来自客户机的远程调用的一个或多个传输。如果您设置的服务器只有一个传输，那么您可以向RcfServer构造函数提供相应的RCF::Endpoint参数：
```cpp
RCF::RcfServer server( RCF::TcpEndpoint(50001) );
```
或者，要配置多个传输，可以使用RCF::RcfServer::addEndpoint()：
```cpp
RCF::RcfServer server;
server.addEndpoint( RCF::TcpEndpoint(50001) );
server.addEndpoint( RCF::UdpEndpoint(50002) );
```
要执行额外的传输相关配置，可以从RCF::RcfServer::addEndpoint()获取返回值：
```cpp
RCF::RcfServer server;
RCF::ServerTransport& tcpTransport = server.addEndpoint(RCF::TcpEndpoint(50001));
tcpTransport.setConnectionLimit(100);
tcpTransport.setMaxIncomingMessageLength(1024 * 1024);
```

添加服务绑定
服务对象负责应用程序的实际服务器功能。当RcfServer接收到来自客户机的远程调用请求时，它使用请求中指定的服务绑定名称来定位相关的服务绑定，然后将远程调用分派到该服务对象上的相关函数。

服务对象总是绑定到RCF接口。要创建服务绑定，请使用RCF::RcfServer::bind<>()。服务器上的每个绑定都由其绑定名称标识，该名称通常是绑定中使用的RCF接口的运行时名称。

所以代码如下：
```cpp
server.bind<I_PrintService>(printService);
```
，创建具有服务绑定名称“I_PrintService”的服务绑定。

服务绑定名称也可以显式设置：
```cpp
server.bind<I_PrintService>(printService, "CustomBindingName");
```
来自客户机的每个远程调用请求都包含一个服务绑定名称，服务器使用该名称来分派远程调用。RCF客户机提供的缺省服务绑定名称是它使用的RCF接口的运行时名称。所以客户端代码如下：
```cpp
RcfClient<I_PrintService> client( RCF::TcpEndpoint(50001) );
client.Print("Hello World");
```
，进行远程调用，服务器将远程调用发送给具有服务绑定名称“I_PrintService”的服务对象。

客户端还可以显式设置服务绑定名称：
```cpp
RcfClient<I_PrintService> client( RCF::TcpEndpoint(50001), "CustomBindingName" );
client.Print("Hello World");
```
，将调用发送到指定的服务绑定。

启动和停止服务器
在调用RCF::RcfServer::start()之前，RcfServer实例不会开始调度远程调用。

可以通过调用RCF::RcfServer::stop()手动停止服务器：
```cpp
RCF::RcfServer server( RCF::TcpEndpoint(50001) );
server.start();
// ...
server.stop();
```
如果RcfServer对象超出范围，服务器将自动停止。

服务器线程
默认情况下，RcfServer将使用一个线程在其所有传输之间分派调用。可以通过显式地将线程池分配给RcfServer来修改此行为：
```cpp
RCF::RcfServer server( RCF::TcpEndpoint(50001) );
RCF::ThreadPoolPtr threadPoolPtr( new RCF::ThreadPool(1, 5) );
server.setThreadPool(threadPoolPtr);
```
ThreadPool可以配置为使用固定数量的线程，也可以根据服务器负载使用不同数量的线程：
```cpp
// Thread pool with a fixed number of threads (5).
RCF::ThreadPoolPtr threadPoolPtr( new RCF::ThreadPool(5) );
server.setThreadPool(threadPoolPtr);
```
```cpp
// Thread pool with a varying number of threads (1-25).
RCF::ThreadPoolPtr threadPoolPtr( new RCF::ThreadPool(1, 25) );
server.setThreadPool(threadPoolPtr);
```
分配给RcfServer的线程池将由该RcfServer的所有传输共享。

还可以使用RCF::ServerTransport::setThreadPool()将线程池分配给特定的传输：
```cpp
RCF::RcfServer server;
RCF::ThreadPoolPtr tcpThreadPool( new RCF::ThreadPool(1,5) );
RCF::ServerTransport & tcpTransport = server.addEndpoint(RCF::TcpEndpoint(50001));
tcpTransport.setThreadPool(tcpThreadPool);
RCF::ThreadPoolPtr pipeThreadPool( new RCF::ThreadPool(1) );
RCF::ServerTransport & pipeTransport = server.addEndpoint(RCF::Win32NamedPipeEndpoint("SvrPipe"));
pipeTransport.setThreadPool(pipeThreadPool);
```

服务器端会话
RCF为每个到服务器的连接创建一个服务器端RCF::RcfSession对象。RcfSession对象的生命周期与客户机连接的生命周期相匹配，并为服务器端代码提供了一种机制，使其能够跨相同客户机连接上的远程调用持久化状态。

在执行远程调用的服务对象中，您可以通过调用RCF::getCurrentRcfSession()来访问RcfSession：
```cpp
class PrintService
{
public:
    void Print(const std::string & s)
    {
        std::cout << "I_PrintService service: " << s << std::endl;
        RCF::RcfSession & session = RCF::getCurrentRcfSession();
        // ...
    }
};
```
会话对象
会话对象是application c++对象，它们存储在RCF会话中，因此可以在同一连接上的远程调用之间持久化。会话对象的一个典型用例是将应用程序特定的信息与连接关联起来。例如，应用程序逻辑可能要求在客户机连接之后，它应该做的第一件事是调用Login()方法。如果Login()方法成功，则需要持久化连接的已验证状态，以便对同一连接的后续调用可以访问它。

以下功能提供对会话对象的访问：
RCF::RcfSession::createSessionObject<>()
RCF::RcfSession::getSessionObject<>()
RCF::RcfSession::querySessionObject<>()
RCF::RcfSession::deleteSessionObject<>()
会话对象可以是任意类型的，它们的生存期由客户机连接的生存期控制。当客户机连接关闭时，与之关联的任何会话对象都将被销毁。

有关使用会话对象存储身份验证状态的示例，请参见访问控制。

自定义请求和响应数据
正如在客户端编程中所描述的，除了远程调用的参数外，远程调用还可以具有与之关联的额外用户数据。远程调用请求和远程调用响应都可以携带这些数据，您可以使用以下RcfSession函数来访问这些数据：
RCF::RcfSession::getRequestUserData()
RCF::RcfSession::setRequestUserData()
RCF::RcfSession::getResponseUserData()
RCF::RcfSession::setResponseUserData()

会话信息
您可以使用RcfSession查找关于当前客户端连接的许多信息，包括:
所使用的运输工具：
```cpp
RCF::RcfSession & session = RCF::getCurrentRcfSession();
RCF::TransportType transportType = session.getTransportType();
```
客户的网络地址：
```cpp
const RCF::RemoteAddress & clientAddress = session.getClientAddress();
std::string strClientAddress = clientAddress.string();
```
当前请求头：
```cpp
RCF::RemoteCallInfo request = session.getRemoteCallRequest();
```
客户端连接时间：
```cpp
// 根据CRT time()函数的报告，当连接建立时。
time_t connectedAt = session.getConnectedAtTime();
// 连接持续时间(秒)。
std::size_t connectionDurationS = session.getConnectionDuration();
```
此连接上已经进行过多少次调用：
```cpp
std::size_t callsMade = session.getRemoteCallCount();
```
在这个连接上发送和接收了多少字节：
```cpp
std::uint64_t totalBytesReceived = session.getTotalBytesReceived();
std::uint64_t totalBytesSent = session.getTotalBytesSent();
```
此连接上使用的传输协议(参见传输协议)：
```cpp
RCF::TransportProtocol protocol = session.getTransportProtocol();
```
客户端用户名，如果客户端已通过身份验证：
```cpp
std::string clientUserName = session.getClientUserName();
```
客户端提供的SSL证书(如有)：
```cpp
RCF::CertificatePtr clientCertPtr = session.getClientCertificatePtr();
```
有关更多信息，请参阅RCF::RcfSession的参考文档。

访问控制
RCF允许您将访问控制应用于服务器上的单个服务绑定。访问控制作为用户定义的回调函数实现，在该函数中，您可以应用特定于应用程序的逻辑来确定是否应该允许客户机连接访问特定的服务绑定。

例如：
```cpp
bool onPrintServiceAccess(int methodId)
{
    // Return true to allow access, and false to deny access.
    // ...
    return true;
}
```
```cpp
PrintService printService;
RCF::RcfServer server( RCF::TcpEndpoint(50001) );
RCF::ServerBindingPtr bindingPtr = server.bind<I_PrintService>(printService);
auto accessControl = [&](int methodId) { return onPrintServiceAccess(methodId); };
bindingPtr->setAccessControl(accessControl);
server.start();
```
每当客户机试图调用该服务绑定上的方法时，RcfServer将调用访问控制回调。

从访问控制回调函数中，您可以检查当前会话，并确定是否应该授予它对服务的访问权。一旦认证被授予，您可能希望将认证状态存储在会话对象中，以便在随后的调用中很容易重用：
```cpp
// App-specific authentication state.
class PrintServiceAuthenticationState
{
public:
    PrintServiceAuthenticationState() : mAuthenticated(false)
    {
    }
    bool            mAuthenticated;
    std::string     mClientUsername;
};
// Servant binding access control.
bool onPrintServiceAccess(int methodId)
{
    RCF::RcfSession & session = RCF::getCurrentRcfSession();
    PrintServiceAuthenticationState & authState = session.getSessionObject<PrintServiceAuthenticationState>(true);
    if (!authState.mAuthenticated)
    {
        // Here we are checking that the client is using either NTLM or Kerberos.
        RCF::TransportProtocol tp =  session.getTransportProtocol();
        if (tp == RCF::Tp_Ntlm || tp == RCF::Tp_Kerberos)
        {
            authState.mAuthenticated = true;
            authState.mClientUsername = session.getClientUserName();
        }
    }
    return authState.mAuthenticated;
}
```
本例使用访问控制回调来检查客户机正在使用的传输协议，并使用该协议来确定客户机的标识。

在某些情况下，您可能希望客户端提供额外的身份验证信息，而不是通过传输协议提供的信息。这通常意味着在接口上有一个等效的Login()方法，需要在接口上的任何其他方法之前调用该方法。访问控制回调可用于验证在调用任何其他方法之前调用Login()：
```cpp
// App-specific login info.
class LoginInfo
{
public:
    // ...
    void serialize(SF::Archive & ar) 
    {}
};
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Login, const LoginInfo &)
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
// App-specific authentication state.
class PrintServiceAuthenticationState
{
public:
    PrintServiceAuthenticationState() : mAuthenticated(false)
    {
    }
    bool            mAuthenticated;
    std::string     mClientUsername;
    LoginInfo       mLoginInfo;
};
// Servant object.
class PrintService
{
public:
    void Login(const LoginInfo & loginInfo)
    {
        RCF::RcfSession & session = RCF::getCurrentRcfSession();
        PrintServiceAuthenticationState & authState = session.getSessionObject<PrintServiceAuthenticationState>(true);
        if (!authState.mAuthenticated)
        {
            RCF::TransportProtocol tp =  session.getTransportProtocol();
            if (tp == RCF::Tp_Ntlm || tp == RCF::Tp_Kerberos)
            {
                authState.mAuthenticated = true;
                authState.mClientUsername = session.getClientUserName();
                authState.mLoginInfo = loginInfo;
            }
        }
    }
    void Print(const std::string & s)
    {
        std::cout << "I_PrintService service: " << s << std::endl;
    }
};
// Servant binding access control.
bool onPrintServiceAccess(int methodId)
{
    if (methodId == 0)
    {
        // Calls to Login() are always allowed through.
        return true;
    }
    else
    {
        RCF::RcfSession & session = RCF::getCurrentRcfSession();
        PrintServiceAuthenticationState & authState = session.getSessionObject<PrintServiceAuthenticationState>(true);
        return authState.mAuthenticated;
    }
}
```
在本例中，访问控制回调允许调用Login()，而对于I_PrintService接口上的任何其他方法，它检查Login()调用创建的身份验证状态是否存在。

Login()方法通过其方法ID在访问控制回调中标识——在本例中为0，因为它是接口上的第一个方法。方法ID是按增量顺序分配的，从0开始，因此，如果Login()是接口上的第三个方法，那么它的方法ID应该是2。

服务器对象
服务器对象是由服务器端代码创建并存储在RcfServer中的特定于应用程序的对象。与会话对象(存储在RcfSession中)不同，服务器对象是持久的，并且可以在创建它们的RCF会话之外访问。服务器对象的生存期由RCF::RcfServer管理，使用垃圾收集策略。

使用以下功能操作服务器对象：
RCF::RcfServer::getServerObject<>()
RCF::RcfServer::queryServerObject<>()
RCF::RcfServer::deleteServerObject<>()
下面是我们如何为经过身份验证的用户创建和更新服务器范围的会话对象：
```cpp
class PrintServiceSession
{
public:
    std::string mUsername;
    std::uint32_t mMessagesPrinted = 0;
};
typedef std::shared_ptr<PrintServiceSession> PrintServiceSessionPtr;
// Servant object.
class PrintService
{
public:
    void Print(const std::string & s)
    {
        std::cout << "I_PrintService service: " << s << std::endl;
        RCF::RcfSession & session = RCF::getCurrentRcfSession();
        RCF::RcfServer & server = session.getRcfServer();
        std::string clientUsername = session.getClientUserName();
        if ( clientUsername.size() > 0 )
        {
            std::uint32_t gcTimeoutMs = 3600 * 1000;
            PrintServiceSessionPtr printSessionPtr = server.getServerObject<PrintServiceSession>(
                clientUsername, 
                gcTimeoutMs);
            if ( printSessionPtr->mUsername.empty() )
            {
                printSessionPtr->mUsername = clientUsername;
            }
            ++printSessionPtr->mMessagesPrinted;
        }
    }
};
```
此代码为每个经过身份验证的用户创建PrintServiceSession对象。PrintServiceSession对象将在与该经过身份验证的用户关联的所有连接之间共享。