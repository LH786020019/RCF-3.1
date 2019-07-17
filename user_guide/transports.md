<!--
 * @Author: haoluo
 * @Date: 2019-07-15 16:48:00
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-17 11:09:44
 * @Description: file content
 -->
## 传输(Transports)
### 1. 传输访问(Transport Access)
Server 和 client 传输负责网络消息的实际传输和接收。一个 `RcfServer` 包含一个或多个 server 传输，而一个 `client` 只包含一个 client 传输。

要访问 `RcfServer` 的 server 传输，捕获 `RcfServer::addEndpoint()` 的返回值：
```cpp
    RCF::RcfServer server;
    RCF::ServerTransport & serverTransportTcp = server.addEndpoint( 
        RCF::TcpEndpoint(50001) );
    RCF::ServerTransport & serverTransportUdp = server.addEndpoint( 
        RCF::UdpEndpoint(50002) );
    server.start();
```
或者，如果 `RcfServer` 只有一个 server 传输，您可以通过调用 [RCF::RcfServer::getServerTransport()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#a4ce23dacf5070ffb1f1689dca8360fe9) 来访问它：
```cpp
    RCF::RcfServer server( RCF::TcpEndpoint(50001) );
    RCF::ServerTransport & serverTransport = server.getServerTransport();
```
在 client 端， client 传输可以通过 [RCF::ClientStub::getTransport()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#aed25109a12fb5eff358971769a9a772d) 来访问它：
```cpp
    RcfClient<I_Echo> client( RCF::TcpEndpoint(50001) );
    RCF::ClientTransport & clientTransport = client.getClientStub().getTransport();
```
### 2. 传输配置(Transport Configuration)
#### 2.1 最大传入消息长度
对于 server 端传输，通常需要设置传入网络消息大小的上限。如果没有上限，格式不正确的请求可能导致 server 上任意大小的内存分配。

一个 RCF server 传输的最大传入消息长度设置默认为 1Mb，可以通过调用 [RCF::ServerTransport::setMaxIncomingMessageLength()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_server_transport.html#ab6252f3ea4dbcabe7a5fb00df0107b52) 来更改：
```cpp
    RCF::RcfServer server( RCF::TcpEndpoint(0) );
    // 将最大消息长度设置为 5Mb
    server.getServerTransport().setMaxIncomingMessageLength(5*1024*1024);
```
类似地， client 传输也有一个最大传入消息长度设置。它默认为 1Mb，可以通过调用 [RCF::ClientTransport::setMaxIncomingMessageLength()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_transport.html#a11aa2132b74185fe2aa6b257fefb3bc4) 来更改：
```cpp
    RcfClient<I_Echo> client( RCF::TcpEndpoint(123) );
    // 将最大消息长度设置为 5Mb
    client.getClientStub().getTransport().setMaxIncomingMessageLength(5*1024*1024);
```
由于 client 端传输仅从其连接到的对等端接收网络消息，因此不正确数据包的风险不如 server 传输的大。

可以查询一个 `RcfClient<>` 获取发送的最新请求和响应消息的大小：
```cpp
        RcfClient<I_Echo> client( RCF::TcpEndpoint(50001) );
        client.Echo("1234");
        // 检索前一个调用的请求和响应大小。
        RCF::ClientTransport & transport = client.getClientStub().getTransport();
        std::size_t requestSize = transport.getLastRequestSize();
        std::size_t responseSize = transport.getLastResponseSize();
```
#### 2.2 连接限制
设置一个 RCF server 传输的最大同时连接数：
```cpp
    RCF::RcfServer server( RCF::TcpEndpoint(0) );
    // 在任何时候最多允许连接 100 个 client
    server.getServerTransport().setConnectionLimit(100);
```
此设置与 UDP server 传输无关，因为 UDP 协议中没有连接的概念。

#### 2.3 系统端口选择
对于基于 IP 的 server 传输，您可以通过指定 0 作为端口号，允许本地系统自动分配一个 server 端口号。当 server 启动时，系统将找到一个空闲端口并将其分配给 server。端口号可以通过 [RCF::IpServerTransport::getPort()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_ip_server_transport.html#a8039d39a935abdf24f5fade86574cac3) 来获取：
```cpp
    RCF::RcfServer server( RCF::TcpEndpoint(0) );
    server.start();
    int port = server.getIpServerTransport().getPort();
    RcfClient<I_Echo> client(( RCF::TcpEndpoint(port) ));
```
#### 2.4 基于 IP 的访问规则
对于基于 IP 的 server 传输，可以根据 client 的 IP 地址允许或拒绝 client 访问。

要配置用于允许 client 的 IP 规则，请使用 [RCF::IpServerTransport::setAllowIps()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_ip_server_transport.html#a4b976e1fda4673ffcbc45e1b7fdfe25e)：
```cpp
    RCF::RcfServer server( RCF::TcpEndpoint(0) );
    RCF::IpServerTransport & ipTransport = dynamic_cast<RCF::IpServerTransport &>(server.getServerTransport());
    std::vector<RCF::IpRule> rules;
    // Match 11.22.33.* (24 个有意义的位).
    rules.push_back( RCF::IpRule( RCF::IpAddress("11.22.33.0"), 24) );
    ipTransport.setAllowIps(rules);
    server.start();
    // 从与 11.22.33.* 匹配的 IP 地址连接的 client 将被授予访问权限，
    // 所有其他 client 将被拒绝。
```
要配置用于拒绝 client 的 IP 规则，请使用 [RCF::IpServerTransport::setDenyIps()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_ip_server_transport.html#afc8fd7b87643f42bf9425ab458229227)：
```cpp
    RCF::RcfServer server( RCF::TcpEndpoint(0) );
    RCF::IpServerTransport & ipTransport = dynamic_cast<RCF::IpServerTransport &>(server.getServerTransport());
    std::vector<RCF::IpRule> rules;
    // Match 11.*.*.* (8 个有意义的位).
    rules.push_back( RCF::IpRule( RCF::IpAddress("11.0.0.0"), 8) );
    // Match 12.22.*.* (16 个有意义的位).
    rules.push_back( RCF::IpRule( RCF::IpAddress("12.22.0.0"), 16) );
    ipTransport.setDenyIps(rules);
    server.start();
    // 从匹配 11.*.*.* 和 12.22.*.* 的IP地址连接的 client 将被拒绝访问，
    // 所有其他 client 将被允许。
```
#### 2.5 IPv4/IPv6
RCF 同时支持 IPv4 和 IPv6。默认情况下，在 RCF 中启用了 IPv6 支持，但是您可以定义 `RCF_FEATURE_IPV6=0` 来禁用它（ 参见[构建 RCF](https://love2.io/@lh786020019/doc/RCF-3.1/building_RCF/index.md) ）。

例如，要在一个环回 IPv4 连接上运行一个 server 和 client：
```cpp
        // 指定要绑定到的一个显式 IPv4 地址
        RCF::RcfServer server( RCF::TcpEndpoint("127.0.0.1", 50001) );
        server.start();
        // 指定要绑定到的一个显式 IPv4 地址
        RcfClient<I_Echo> client( RCF::TcpEndpoint("127.0.0.1", 50001) );
```
要在一个环回 IPv6 连接上运行一个 server 和 client，请指定 `::1` 而不是 `127.0.0.1`：
```cpp
        // 指定要绑定到的一个显式 IPv6 地址
        RCF::RcfServer server( RCF::TcpEndpoint("::1", 50001) );
        server.start();
        // 指定要绑定到的一个显式 IPv6 地址
        RcfClient<I_Echo> client( RCF::TcpEndpoint("::1", 50001) );
```
RCF 使用 POSIX `getaddrinfo()` 函数来解析 IP 地址。`getaddrinfo()` 可以返回 IPv4 或 IPv6 地址，这取决于本地系统和网络的配置。因此，下面的 client 将使用 IPv4 或 IPv6，这取决于本地系统和网络是如何配置的：
```cpp
        // 将解析为 IPv4 或 IPv6，这取决于系统将 machine.domain 解析成什么
        RcfClient<I_Echo> client( RCF::TcpEndpoint("machine.domain", 50001) );
```
您可以使用 [RCF::IpAddressV4](http://www.deltavsoft.com/doc/class_r_c_f_1_1_ip_address_v4.html) 和 [RCF::IpAddressV6](http://www.deltavsoft.com/doc/class_r_c_f_1_1_ip_address_v6.html) 类强制执行 IPv4 或 IPv6 解析：
```cpp
        // 强制 IPv4 地址解析
        RCF::IpAddressV4 addr_4("machine.domain", 50001);
        RcfClient<I_Echo> client_4(( RCF::TcpEndpoint(addr_4) ));
        // 强制 IPv6 地址解析
        RCF::IpAddressV6 addr_6("machine.domain", 50001);
        RcfClient<I_Echo> client_6(( RCF::TcpEndpoint(addr_6) ));
```
在具有双 IPv4/IPv6 堆栈的机器上，您可能希望您的 server 同时监听 IPv4 和 IPv6 地址。要实现可移植，您应该同时监听 `0.0.0.0` 和 `::0`：
```cpp
        // 在 IPv4 和 IPv6 上监听端口 50001
        RCF::RcfServer server;
        server.addEndpoint( RCF::TcpEndpoint("0.0.0.0", 50001) );
        server.addEndpoint( RCF::TcpEndpoint("::0", 50001) );
        server.start();
```
在一些平台上，只监听 `::0` 就足够了，因为系统将把传入的 IPv4 连接转换为 IPv6 连接，使用一个特殊的 IPv6 地址类。

#### 2.6 用于 Client 的本地地址绑定
当一个 `RcfClient<>` 使用一个基于 IP 的传输连接到一个 server 时，默认行为是允许系统决定使用哪个本地网络接口和本地端口。在某些情况下，您可能希望显式地设置 client 应该绑定到的本地网络接口。

您可以在连接之前调用 [RCF::IpClientTransport::setLocalIp()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_ip_client_transport.html#ace7a4296bb4a0d058d9b4adae2e7b3a0) 来实现这一点：
```cpp
        RcfClient<I_Echo> client( RCF::TcpEndpoint("127.0.0.1", 50001) );
        
        RCF::IpClientTransport & ipTransport = client.getClientStub().getIpTransport();
        // 强制 client 绑定到一个特定的本地网络接口(127.0.0.1)。
        ipTransport.setLocalIp( RCF::IpAddress("127.0.0.1", 0) );
        client.getClientStub().connect();
```
在连接好一个 `RcfClient<>` 之后，您可以通过调用 [RCF::IpClientTransport::getAssignedLocalIp()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_ip_client_transport.html#a2d3d86e21381d09bd200589958eb44ef) 来确定它绑定到哪个本地网络接口和端口：
```cpp
        RcfClient<I_Echo> client( RCF::TcpEndpoint("127.0.0.1", 50001) );
        RCF::IpClientTransport & ipTransport = client.getClientStub().getIpTransport();
        client.getClientStub().connect();
        // 找出 client 绑定到哪个本地网络接口
        RCF::IpAddress localIp = ipTransport.getAssignedLocalIp();
        std::string localInterface = localIp.getIp();
        int localPort = localIp.getPort();
```
#### 2.7 套接字级访问(Socket Level Access)
RCF 提供对 client 和 server 传输的底层 OS 原语（ 如套接字(`sockets`)和句柄(`handles`) ）的访问。例如：
```cpp
        // Client-side.
        RcfClient<I_Echo> client( RCF::TcpEndpoint("127.0.0.1", 50001) );
        client.getClientStub().connect();
        RCF::TcpClientTransport & tcpClientTransport = 
            dynamic_cast<RCF::TcpClientTransport &>( client.getClientStub().getTransport() );
        // 获取 client 套接字句柄
        int sock = tcpClientTransport.getNativeHandle();
```
```cpp
        // Server-side.
        RCF::NetworkSession & networkSession = RCF::getCurrentRcfSession().getNetworkSession();
        
        RCF::TcpNetworkSession & tcpNetworkSession = dynamic_cast<RCF::TcpNetworkSession &>(networkSession);
        // 获取 server 套接字句柄
        int sock = tcpNetworkSession.getNativeHandle();
```
如果需要设置自定义套接字选项，这将非常有用。

### 3. 传输实现(Transport Implementations)
本节介绍 RCF 中实现的各种传输类型。

#### 3.1 TCP
TCP 端点在 RCF 中由 [RCF::TcpEndpoint](http://www.deltavsoft.com/doc/class_r_c_f_1_1_tcp_endpoint.html) 类表示，该类由一个 IP 地址和一个端口号构造。
```cpp
        RCF::RcfServer server( RCF::TcpEndpoint("0.0.0.0", 50001) );
        server.start();
        RcfClient<I_Echo> client( RCF::TcpEndpoint("127.0.0.1", 50001) );
```
Server 传输将 IP 地址解释为要监听的本地网络接口。例如，为了监听所有可用的 IPv4 网络接口，应该指定 `"0.0.0.0"`，而 `"127.0.0.1"` 应该指定为只监听环回 IPv4 接口。如果没有指定 IP 地址，则假设 `"127.0.0.1"`。

#### 3.2 UDP
与 [RCF::TcpEndpoint](http://www.deltavsoft.com/doc/class_r_c_f_1_1_tcp_endpoint.html) 类似，[RCF::UdpEndpoint](http://www.deltavsoft.com/doc/class_r_c_f_1_1_udp_endpoint.html) 由一个 IP 地址和一个端口号构造。
```cpp
        RCF::RcfServer server( RCF::UdpEndpoint("0.0.0.0", 50001) );
        server.start();
        RcfClient<I_Echo> client( RCF::UdpEndpoint("127.0.0.1", 50001) );
```
[RCF::UdpEndpoint](http://www.deltavsoft.com/doc/class_r_c_f_1_1_udp_endpoint.html) 还包含一些额外的功能，用于处理`多播(multicasting)`和`广播(broadcasting)`。

##### 3.2.1 多播(Multicasting)
[RCF::UdpEndpoint](http://www.deltavsoft.com/doc/class_r_c_f_1_1_udp_endpoint.html) 可以配置为监听一个多播 IP 地址：
```cpp
        // 监听多播地址 232.5.5.5，在端口 50001 上，在所有网络接口("0.0.0.0")上。
        RCF::UdpEndpoint udpEndpoint("0.0.0.0", 50001);
        udpEndpoint.listenOnMulticast("232.5.5.5");
        RCF::RcfServer server(udpEndpoint);
        server.start();
```
注意，server 仍然需要指定一个本地网络接口来监听。

要发送多播消息，在创建 client 端时指定一个多播 IP 地址和一个端口号：
```cpp
            RcfClient<I_Echo> client( RCF::UdpEndpoint("232.5.5.5", 50001) );
            client.Echo(RCF::Oneway, "ping");
```
##### 3.2.2 广播(Broadcasting)
要发送广播消息，请指定一个广播 IP 地址和一个端口号：
```cpp
            RcfClient<I_Echo> client( RCF::UdpEndpoint("255.255.255.255", 50001) );
            client.Echo(RCF::Oneway, "ping");
```
##### 3.2.3 地址共享
RCF 的 UDP server 传输可以配置为共享其地址绑定，这样多个 RcfServer 就可以在同一接口的同一端口上侦听。默认情况下，监听多播地址时启用此功能，但也可以在侦听非多播地址时启用。如果同一台机器上的多个进程需要监听相同的广播，这将非常有用：
```cpp
        EchoImpl echoImpl;
        RCF::UdpEndpoint udpEndpoint1("0.0.0.0", 50001);
        udpEndpoint1.enableSharedAddressBinding();
        RCF::RcfServer server1(udpEndpoint1);
        server1.bind<I_Echo>(echoImpl);
        server1.start();
        RCF::UdpEndpoint udpEndpoint2("0.0.0.0", 50001);
        udpEndpoint2.enableSharedAddressBinding();
        RCF::RcfServer server2(udpEndpoint2);
        server2.bind<I_Echo>(echoImpl);
        server2.start();
        // 此广播消息将被两个 server 接收
        RcfClient<I_Echo> client( RCF::UdpEndpoint("255.255.255.255", 50001) );
        client.Echo(RCF::Oneway, "ping");
```
##### 3.2.4 Server 发现
在 server 是在动态分配的端口上启动的情况下，多播和广播是向 client 通信 server IP 地址和端口的一种有用方法。例如：
```cpp
// 用于广播一个 TCP server 端口号的接口
RCF_BEGIN(I_Broadcast, "I_Broadcast")
    RCF_METHOD_V1(void, ServerIsRunningOnPort, int)
RCF_END(I_Broadcast)
// 用于接收 I_Broadcast 消息的实现类
class BroadcastImp{
public:
    BroadcastImpl() : mPort(){}
    void ServerIsRunningOnPort(int port){
        mPort = port;
    }
    int mPort;
};
// 一个 server 线程运行此函数，用于每秒广播 server 位置一次。
void broadcastThread(int port, const std::string &multicastIp, int multicastPort)
{
    RcfClient<I_Broadcast> client( RCF::UdpEndpoint(multicastIp, multicastPort) );
    client.getClientStub().setRemoteCallMode(RCF::Oneway);
    // 每秒广播一条消息。
    while (true) {
        client.ServerIsRunningOnPort(port);
        RCF::sleepMs(1000);
    }
}
```
```cpp
        // ***** Server side ****
        // 在一个动态分配的端口上启动一个 server
        EchoImpl echoImpl;
        RCF::RcfServer server( RCF::TcpEndpoint(0));
        server.bind<I_Echo>(echoImpl);
        server.start();
        // 检索端口号
        int port = server.getIpServerTransport().getPort();        
        // 开始广播端口号
        RCF::ThreadPtr broadcastThreadPtr(new RCF::Thread(
            [=]() { broadcastThread(port, "232.5.5.5", 50001); }));

        // ***** Client side ****
        // Client 在做任何其他事情之前都会先收听广播
        RCF::UdpEndpoint udpEndpoint("0.0.0.0", 50001);
        udpEndpoint.listenOnMulticast("232.5.5.5");
        RCF::RcfServer clientSideBroadcastListener(udpEndpoint);
        BroadcastImpl broadcastImpl;
        clientSideBroadcastListener.bind<I_Broadcast>(broadcastImpl);
        clientSideBroadcastListener.start();
        // 等待一个广播消息
        while (!broadcastImpl.mPort) {
            RCF::sleepMs(1000);
        }
        // 一旦 client 知道端口号，它们就可以连接了。
        RcfClient<I_Echo> client( RCF::TcpEndpoint(broadcastImpl.mPort));
        client.Echo("asdf");
```
注意，这里我们实际上使用一个多播地址向 client 广播信息。如果在这个特定的网络上无法进行多播，我们也可以使用 IP 广播地址而不是 IP 多播地址。

#### 3.3 Win32 命名管道(Win32 Named Pipes)
RCF 支持 Win32 命名管道传输。[RCF::Win32NamedPipeEndpoint](http://www.deltavsoft.com/doc/class_r_c_f_1_1_win32_named_pipe_endpoint.html) 接受一个构造函数参数，它是命名管道的名称，带或不带前导 `\\.\pipe\` 前缀。
```cpp
        RCF::RcfServer server( RCF::Win32NamedPipeEndpoint("PrintSvrPipe") );
        server.start();
        RcfClient<I_Echo> client( RCF::Win32NamedPipeEndpoint("PrintSvrPipe") );
```
使用 Win32 命名管道的一个优点是，它们允许对 client 进行简单的身份验证。使用 Win32 命名管道 server 传输的一个 server 可以通过 [RCF::Win32NamedPipeImpersonator](http://www.deltavsoft.com/doc/class_r_c_f_1_1_win32_named_pipe_impersonator.html) 类对其 client 进行身份验证，该类使用 Windows API 函数 `ImpersonateNamedPipeClient()` 来模拟 client：
```cpp
RCF_BEGIN(I_Echo, "I_Echo")
    RCF_METHOD_R1(std::string, Echo, const std::string &)
RCF_END(I_Echo)
class EchoImpl{
public:
    std::string Echo(const std::string & s){
        // Impersonate(模拟) client
        RCF::Win32NamedPipeImpersonator impersonator(RCF::getCurrentRcfSession());
        std::cout << "Client user name: " << RCF::getMyUserName();
        return s;
    }
};
```
如果我们使用一个到 `127.0.0.1` 的 TCP 连接来替代上面的 Win32 命名管道，则需要启用 `Kerberos` 或 `NTLM` 身份验证来安全地确定 client 用户名（ 请参阅[传输协议](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/transports_protocols.md) ）。

#### 3.4 UNIX 域套接字(UNIX Domain Sockets)
UNIX 域套接字的功能类似于 Win32 命名管道，它允许在同一台机器上的 server 和 client 之间进行有效的通信。[RCF::UnixLocalEndpoint](http://www.deltavsoft.com/doc/class_r_c_f_1_1_unix_local_endpoint.html) 接受一个参数，它是 UNIX 域套接字的名称。此名称必须是一个有效的文件系统路径。对于 server ，程序必须具有创建给定路径的足够权限，并且文件必须不存在。对于 client，程序必须具有访问给定路径的足够权限。

下面是一个简单的例子：
```cpp
    RCF::RcfServer server( RCF::UnixLocalEndpoint("/home/xyz/MySocket"));
    server.start();
    RcfClient<I_Echo> client( RCF::UnixLocalEndpoint("/home/xyz/MySocket"));
```
#### 3.5 HTTP/HTTPS
RCF 支持通过 HTTP 和 HTTPS 协议进行隧道化远程调用( `tunneling remote call` )。特别是，远程调用可以通过 HTTP 和 HTTPS 代理进行定向。

HTTPS 本质上是在 SSL 协议之上分层的 HTTP 协议。因此，HTTPS 的 SSL 方面的配置与 SSL 传输协议（ 请参阅[传输协议](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/transports_protocols.md) ）的配置方式相同。

##### 3.5.1 Server 端
要使用一个 HTTP 端点设置一个 server，请使用 [RCF::HttpEndpoint](http://www.deltavsoft.com/doc/class_r_c_f_1_1_http_endpoint.html)：
```cpp
        RCF::RcfServer server( RCF::HttpEndpoint("0.0.0.0", 80) );
        PrintService printService;
        server.bind<I_PrintService>(printService);
        server.start();
```
类似地，对于一个 HTTPS 端点, 请使用 [RCF::HttpsEndpoint](http://www.deltavsoft.com/doc/class_r_c_f_1_1_https_endpoint.html)：
```cpp
        RCF::RcfServer server( RCF::HttpsEndpoint("0.0.0.0", 443) );
        PrintService printService;
        server.bind<I_PrintService>(printService);
        server.setCertificate( RCF::CertificatePtr( new RCF::PfxCertificate(
            "C:\\serverCert.p12", 
            "password", 
            "CertificateName") ) );
        server.start();
```
##### 3.5.2 Client 端
Client 端配置类似，对于一个 HTTP client，请使用 [RCF::HttpEndpoint](http://www.deltavsoft.com/doc/class_r_c_f_1_1_http_endpoint.html)：
```cpp
        RcfClient<I_PrintService> client( RCF::HttpEndpoint("printsvr.acme.com", 80) );
        client.Print("Hello World");
```
，对于一个 HTTP client，请使用 [RCF::HttpsEndpoint](http://www.deltavsoft.com/doc/class_r_c_f_1_1_https_endpoint.html)：
```cpp
        RcfClient<I_PrintService> client( RCF::HttpsEndpoint("printsvr.acme.com", 443) );
        client.getClientStub().setCertificateValidationCallback(&schannelValidate);
        client.Print("Hello World");
```
最后，要通过一个 HTTP 或 HTTPS 代理直接进行远程调用，请使用 [RCF::ClientStub::setHttpProxy()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#ac347fc281307ef5506ceefdaec5eab19) 和 [RCF::ClientStub::setHttpProxyPort()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#ae37efe5cf75d7eefd6c7b88c1e0a924d) 函数：
```cpp
            client.getClientStub().setHttpProxy("web-proxy.acme.com");
            client.getClientStub().setHttpProxyPort(8080);
            client.Print("Hello World");
```
##### 3.5.3 反向代理
HTTP 反向代理在 Internet 上很常见，用于为后端(back-end) HTTP server 提供负载平衡(load balancing)和 SSL 卸载(offloadin)等功能。

反向代理通常对 client 和目标 server 是透明的。您可以在 RCF client 和 server 之间放置反向代理，这为在一组 RCF server 之间分配负载提供了开箱即用的机制。

在负载平衡场景中，您可能希望 RCF client 继续向它最初连接到的后端 server 发送请求。为此，您需要使用支持`会话关联(session affinity)`的反向代理。会话关联通常由反向代理实现，方法是在通信流的初始 HTTP 响应中插入一个特殊的 `HTTP cookie`。然后，RCF client 将在后续请求中自动包含此 cookie，从而允许反向代理将后续请求路由到相同的后端 server。

如果您有多个 RCF client 连接，您可以将它们配置为连接到相同的后端 server，方法是从第一个连接检索 HTTP cookie，然后将其应用于其他连接（ 请参阅 [RCF::ClientStub::setHttpCookies()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a05c0fb1bf52fa5804fc2820b2c66e501) ）。

RCF中 可以使用的另一个反向代理特性是 SSL 卸载。通过 SSL 卸载, RCF client 使用 HTTPS 连接进行连接，但是 HTTPS 连接由反向代理解密，解密后的 HTTP 流被转发到后端 server。这允许 client 端和后端 server 之间进行安全通信，而反向代理承担 SSL 加密/解密的负载。这还具有将证书配置从多个后端 server 集中到反向代理 server 的优势。

RCF 已使用下面的反向代理测试过：
- [IIS ARR](https://www.iis.net/downloads/microsoft/application-request-routing)
- [nginx](https://www.nginx.com/)