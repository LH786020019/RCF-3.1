<!--
 * @Author: haoluo
 * @Date: 2019-07-15 11:50:47
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-17 18:44:40
 * @Description: file content
 -->
## 服务器端编程(Server-side Programming)
在 server 端，[RCF::RcfServer](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html) 是基本类，它负责派遣来自 client 的远程调用。一个 `RcfServer` 包含一个或多个传输，它侦听来自 client 的远程调用。`RcfServer` 还公开一个或多个服务绑定，它将来自 client 的远程调用派遣给这些绑定。

[教程](https://love2.io/@lh786020019/doc/RCF-3.1/tutorial/index.md)展示了一个 server 的最小示例。首先，使用 TCP 传输创建一个 RcfServer：
```cpp
RCF::RcfServer server( RCF::TcpEndpoint(50001) );
```
然后为 `I_PrintService` 接口配置一个服务绑定：
```cpp
PrintService printService;
server.bind<I_PrintService>(printService);
```
最后，server 启动并开始响应来自 client 的远程调用：
```cpp
server.start();
```

### 1. 配置一个 Server
#### 1.1 添加传输
一个 `RcfServer` 侦听来自客户机的远程调用的一个或多个传输。如果您设置的 server 只有一个传输，那么您可以向 `RcfServer` 构造函数提供相应的 [RCF::Endpoint](http://www.deltavsoft.com/doc/class_r_c_f_1_1_endpoint.html) 参数：
```cpp
RCF::RcfServer server( RCF::TcpEndpoint(50001) );
```
或者，要配置多个传输，可以使用 [RCF::RcfServer::addEndpoint()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#a431467ee38910f4797080db9e3f30986)：
```cpp
RCF::RcfServer server;
server.addEndpoint( RCF::TcpEndpoint(50001) );
server.addEndpoint( RCF::UdpEndpoint(50002) );
```
要执行额外的传输相关的配置，可以从 [RCF::RcfServer::addEndpoint()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#a431467ee38910f4797080db9e3f30986) 获取返回值：
```cpp
RCF::RcfServer server;
RCF::ServerTransport& tcpTransport = server.addEndpoint(RCF::TcpEndpoint(50001));
tcpTransport.setConnectionLimit(100);
tcpTransport.setMaxIncomingMessageLength(1024 * 1024);
```

#### 2. 添加服务绑定
服务对象(Servant object)负责应用程序的实际 server 功能。当一个 `RcfServer` 接收到来自一个 client 的远程调用请求时，它使用请求中指定的服务绑定名称来定位相关的服务绑定，然后将远程调用分派到该服务对象上的相关函数。

服务对象总是绑定到一个 RCF 接口。要创建一个服务绑定，请使用 [RCF::RcfServer::bind<>()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#a3e8f6a060a87beb96b5690e9829c9c3f) 。Server 上的每个绑定都由其绑定名称标识，该名称通常是绑定中使用的 RCF 接口的运行时名称。

所以对于下面的 server 端代码：
```cpp
server.bind<I_PrintService>(printService);
```
，创建一个具有服务绑定名称 `"I_PrintService"` 的服务绑定。

服务绑定名称也可以显式设置：
```cpp
server.bind<I_PrintService>(printService, "CustomBindingName");
```
来自一个 client 的每个远程调用请求都包含一个服务绑定名称，server 使用该名称来派遣远程调用。一个 RCF client 提供的默认服务绑定名称是它使用的 RCF 接口的运行时名称。所以对于下面的 client 端代码：
```cpp
RcfClient<I_PrintService> client( RCF::TcpEndpoint(50001) );
client.Print("Hello World");
```
，将会进行一次远程调用，server 将远程调用派遣给具有服务绑定名称 `"I_PrintService"` 的一个服务对象。

Client 还可以显式设置服务绑定名称：
```cpp
RcfClient<I_PrintService> client( RCF::TcpEndpoint(50001), "CustomBindingName" );
client.Print("Hello World");
```
，将调用派遣给指定的服务绑定。

#### 1.3 启动和停止 Server
在调用 [RCF::RcfServer::start()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#a737d7cf9860b1d49209b2ebe891e4449) 之前，一个 `RcfServer` 实例将不会开始派遣远程调用。

可以通过调用 [RCF::RcfServer::stop()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#acda4490f988f83af77528e3b5b5074f5) 手动停止 server：
```cpp
RCF::RcfServer server( RCF::TcpEndpoint(50001) );
server.start();
// ...
server.stop();
```
如果 `RcfServer` 对象超出范围，server 将自动停止。

#### 1.4 服务器线程(Server Threading)
默认情况下，`RcfServer` 将使用单个线程在其所有传输之间派遣调用。可以通过显式地将一个线程池分配给 `RcfServer` 来修改此行为：
```cpp
RCF::RcfServer server( RCF::TcpEndpoint(50001) );
RCF::ThreadPoolPtr threadPoolPtr( new RCF::ThreadPool(1, 5) );
server.setThreadPool(threadPoolPtr);
```
[RCF::ThreadPool](http://www.deltavsoft.com/doc/class_r_c_f_1_1_thread_pool.html) 可以配置为使用固定数量的线程，也可以根据 server 负载使用不同数量的线程：
```cpp
// 具有固定数量(5)线程的线程池
RCF::ThreadPoolPtr threadPoolPtr( new RCF::ThreadPool(5) );
server.setThreadPool(threadPoolPtr);
```
```cpp
// 具有不同数量(1-25)线程的线程池
RCF::ThreadPoolPtr threadPoolPtr( new RCF::ThreadPool(1, 25) );
server.setThreadPool(threadPoolPtr);
```
分配给 `RcfServer` 的一个线程池将由该 `RcfServer` 的所有传输共享。

您还可以使用 [RCF::ServerTransport::setThreadPool()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_server_transport.html#a035886bcaf26a27f33ae201eb6410fee) 将线程池分配给特定的传输：
```cpp
RCF::RcfServer server;
RCF::ThreadPoolPtr tcpThreadPool( new RCF::ThreadPool(1,5) );
RCF::ServerTransport & tcpTransport = server.addEndpoint(RCF::TcpEndpoint(50001));
tcpTransport.setThreadPool(tcpThreadPool);
RCF::ThreadPoolPtr pipeThreadPool( new RCF::ThreadPool(1) );
RCF::ServerTransport & pipeTransport = server.addEndpoint(RCF::Win32NamedPipeEndpoint("SvrPipe"));
pipeTransport.setThreadPool(pipeThreadPool);
```

### 2. 服务器端会话(Server-side Sessions)
RCF 为每个到 server 的连接创建一个 server 端 [RCF::RcfSession](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_session.html) 对象。`RcfSession` 对象的生命周期与 client 连接的生命周期相匹配，并为 server 端代码提供了一种机制，使其能够在同一 client 连接上的远程调用之间持久化状态。

在执行远程调用的一个服务对象中，您可以通过调用 [RCF::getCurrentRcfSession()](http://www.deltavsoft.com/doc/_rcf_session_8hpp.html#a9d9e37f6e66bcbc3a1f6e5775d107cb2) 来访问 `RcfSession`：
```cpp
class PrintService{
public:
    void Print(const std::string & s){
        std::cout << "I_PrintService service: " << s << std::endl;
        RCF::RcfSession & session = RCF::getCurrentRcfSession();
        // ...
    }
};
```

#### 2.1 会话对象(Session Objects)
会话对象是应用程序 C++ 对象，它们存储在 RCF 会话中，因此可以在同一连接上的远程调用之间持久化。会话对象的一个典型用例是将应用程序特定的信息与一个连接关联起来。例如，应用程序逻辑可能要求在一个 client 连接之后，它应该做的第一件事是调用 `Login()` 方法。如果 `Login()` 方法成功，则需要持久化连接的已验证状态，以便对同一连接的后续调用可以访问它。

以下函数提供对会话对象的访问：
- [RCF::RcfSession::createSessionObject<>()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_session.html#a1632a13e375dcaabb28cc8df3e7926f3)
- [RCF::RcfSession::getSessionObject<>()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_session.html#ae0de2d50513e2e7c75c8363352430fa0)
- [RCF::RcfSession::querySessionObject<>()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_session.html#a4ff730122e48a7fb538b26042bd85914)
- [RCF::RcfSession::deleteSessionObject<>()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_session.html#afd7f54e613cc9a1f9892cf19a4d4e0b4)

会话对象可以是任意类型的，它们的生命周期由 client 连接的生命周期控制。当 client 连接关闭时，与之关联的任何会话对象都将被销毁。

有关使用会话对象存储身份验证状态的一个示例，请参见 `第3章：访问控制`。

#### 2.2 自定义请求和响应数据
正如在[客户端编程](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/client-side_programming.md)中所描述的，除了远程调用的参数外，远程调用还可以具有与之关联的额外用户数据。远程调用请求和远程调用响应都可以携带这些数据，您可以使用以下 `RcfSession` 函数来访问这些数据：
- [RCF::RcfSession::getRequestUserData()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_session.html#ad3c4e6aced6599b1bb919e4a88666875)
- [RCF::RcfSession::setRequestUserData()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_session.html#a788d0d1b28905914c89ca64d4ada14e6)
- [RCF::RcfSession::getResponseUserData()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_session.html#a31f0b39ffb0031b345f2e0b083da6296)
- [RCF::RcfSession::setResponseUserData()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_session.html#a2896d673a3c8bb40365b27acb728ad08)

#### 2.3 会话信息
您可以使用 `RcfSession` 查找关于当前 client 连接的许多信息，包括:
- 所使用的传输：
    ```cpp
    RCF::RcfSession & session = RCF::getCurrentRcfSession();
    RCF::TransportType transportType = session.getTransportType();
    ```
- Client 的网络地址：
    ```cpp
    const RCF::RemoteAddress & clientAddress = session.getClientAddress();
    std::string strClientAddress = clientAddress.string();
    ```
- 当前请求头：
    ```cpp
    RCF::RemoteCallInfo request = session.getRemoteCallRequest();
    ```
- Client 已经连接的时间：
    ```cpp
    // 当连接建立时，由 CRT time() 函数报告
    time_t connectedAt = session.getConnectedAtTime();
    // 连接持续时间(秒)
    std::size_t connectionDurationS = session.getConnectionDuration();
    ```
- 在此连接上已经进行过多少次调用：
    ```cpp
    std::size_t callsMade = session.getRemoteCallCount();
    ```
- 在此连接上发送和接收了多少字节：
    ```cpp
    std::uint64_t totalBytesReceived = session.getTotalBytesReceived();
    std::uint64_t totalBytesSent = session.getTotalBytesSent();
    ```
- 在此连接上使用的传输协议（ 请参阅[传输协议](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/transports_protocols.md) )：
    ```cpp
    RCF::TransportProtocol protocol = session.getTransportProtocol();
    ```
- Client 用户名，如果 client 已通过身份验证：
    ```cpp
    std::string clientUserName = session.getClientUserName();
    ```
- Client 提供的 SSL 证书（如果有的话）：
    ```cpp
    RCF::CertificatePtr clientCertPtr = session.getClientCertificatePtr();
    ```

有关更多信息，请参阅 [RCF::RcfSession](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_session.html) 的参考文档。

### 3. 访问控制
RCF 允许您将访问控制应用于 server 上的单个服务绑定。访问控制作为一个用户定义的回调函数实现，在该函数中，您可以应用应用程序特定的逻辑来确定是否应该允许一个 client 连接访问一个特定的服务绑定。

例如：
```cpp
bool onPrintServiceAccess(int methodId){
    // 返回 true 以允许访问，返回 false 以拒绝访问。
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
每当 client 试图调用该服务绑定上的方法时，`RcfServer` 将调用访问控制回调函数。

从访问控制回调函数中，您可以检查当前会话，并确定是否应该授予它对服务的访问权。一旦认证被授予，您可能希望将认证状态存储在一个会话对象中，以便在随后的调用中很容易重用：
```cpp
// 应用程序特定的身份验证状态
class PrintServiceAuthenticationState{
public:
    PrintServiceAuthenticationState() : mAuthenticated(false){}
    bool            mAuthenticated;
    std::string     mClientUsername;
};
// 服务绑定访问控制
bool onPrintServiceAccess(int methodId){
    RCF::RcfSession & session = RCF::getCurrentRcfSession();
    PrintServiceAuthenticationState & authState = session.getSessionObject<PrintServiceAuthenticationState>(true);
    if (!authState.mAuthenticated) {
        // 这里我们检查客户机是否使用 NTLM 或 Kerberos
        RCF::TransportProtocol tp =  session.getTransportProtocol();
        if (tp == RCF::Tp_Ntlm || tp == RCF::Tp_Kerberos) {
            authState.mAuthenticated = true;
            authState.mClientUsername = session.getClientUserName();
        }
    }
    return authState.mAuthenticated;
}
```
本例使用访问控制回调函数来检查 client 正在使用的传输协议，并使用该协议来确定 client 的标识。

在某些情况下，您可能希望 client 提供额外的身份验证信息，而不是通过传输协议提供的信息。这通常意味着在接口上有一个等效的 `Login()` 方法，需要在接口上的任何其他方法之前调用该方法。访问控制回调函数可用于验证在调用任何其他方法之前调用 `Login()`：
```cpp
// 应用程序特定的登录信息
class LoginInfo{
public:
    // ...
    void serialize(SF::Archive & ar){}
};
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Login, const LoginInfo &)
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
// 应用程序特定的身份验证状态
class PrintServiceAuthenticationState{
public:
    PrintServiceAuthenticationState() : mAuthenticated(false){}
    bool            mAuthenticated;
    std::string     mClientUsername;
    LoginInfo       mLoginInfo;
};
// 服务对象
class PrintService{
public:
    void Login(const LoginInfo & loginInfo){
        RCF::RcfSession & session = RCF::getCurrentRcfSession();
        PrintServiceAuthenticationState & authState = session.getSessionObject<PrintServiceAuthenticationState>(true);
        if (!authState.mAuthenticated) {
            RCF::TransportProtocol tp =  session.getTransportProtocol();
            if (tp == RCF::Tp_Ntlm || tp == RCF::Tp_Kerberos) {
                authState.mAuthenticated = true;
                authState.mClientUsername = session.getClientUserName();
                authState.mLoginInfo = loginInfo;
            }
        }
    }
    void Print(const std::string & s){
        std::cout << "I_PrintService service: " << s << std::endl;
    }
};
// 服务绑定访问控制
bool onPrintServiceAccess(int methodId){
    if (methodId == 0) {
        // 始终允许对 Login() 的调用
        return true;
    } else {
        RCF::RcfSession & session = RCF::getCurrentRcfSession();
        PrintServiceAuthenticationState & authState = session.getSessionObject<PrintServiceAuthenticationState>(true);
        return authState.mAuthenticated;
    }
}
```
在本例中，访问控制回调函数始终允许对 `Login()` 的调用，而对于 `I_PrintService` 接口上的任何其他方法，它检查 `Login()`` 调用创建的身份验证状态是否存在。

`Login()` 方法通过其方法 ID(在本例中为0) 在访问控制回调函数中标识 ，因为它是接口上的第一个方法。方法 ID 是按增量顺序分配的，从 0 开始，因此，如果 `Login()` 是接口上的第 3 个方法，那么它的方法 ID 应该是 2。

### 4. 服务器对象(Server Objects)
Server 对象是由 server 端代码创建并存储在 `RcfServer` 中的应用程序特定的对象。与会话对象(存储在 `RcfSession` 中)不同，server 对象是持久的，并且可以在创建它们的 RCF 会话之外访问。Server 对象的生命周期由 [RCF::RcfServer](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html) 管理，使用一个垃圾收集策略。

使用以下函数操作 server 对象：
- [RCF::RcfServer::getServerObject<>()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#a98b2f4b6aee9df2c8daf22a047aad01d)
- [RCF::RcfServer::queryServerObject<>()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#af2c3199a5f10051422f7675ea37fff04)
- [RCF::RcfServer::deleteServerObject<>()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#a14e08805f7726f3b5d74e615d1119d7d)

下面是我们如何为经过身份验证的用户创建和更新 server 范围的会话对象：
```cpp
class PrintServiceSession{
public:
    std::string mUsername;
    std::uint32_t mMessagesPrinted = 0;
};
typedef std::shared_ptr<PrintServiceSession> PrintServiceSessionPtr;
// 服务对象
class PrintService{
public:
    void Print(const std::string & s){
        std::cout << "I_PrintService service: " << s << std::endl;
        RCF::RcfSession & session = RCF::getCurrentRcfSession();
        RCF::RcfServer & server = session.getRcfServer();
        std::string clientUsername = session.getClientUserName();
        if ( clientUsername.size() > 0 ) {
            std::uint32_t gcTimeoutMs = 3600 * 1000;
            PrintServiceSessionPtr printSessionPtr = server.getServerObject<PrintServiceSession>(
                clientUsername, 
                gcTimeoutMs);
            if ( printSessionPtr->mUsername.empty() ) {
                printSessionPtr->mUsername = clientUsername;
            }
            ++printSessionPtr->mMessagesPrinted;
        }
    }
};
```
此代码为每个经过身份验证的用户创建 `PrintServiceSession` 对象。`PrintServiceSession` 对象将在与经过身份验证的该用户关联的所有连接之间共享。