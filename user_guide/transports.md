<!--
 * @Author: haoluo
 * @Date: 2019-07-15 16:48:00
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-15 17:00:59
 * @Description: file content
 -->
传输
传输访问
服务器和客户机传输负责网络消息的实际传输和接收。RcfServer包含一个或多个服务器传输，而客户机只包含一个客户机传输。

要访问RcfServer的服务器传输，捕获RcfServer::addEndpoint()的返回值：
```cpp
    RCF::RcfServer server;
    RCF::ServerTransport & serverTransportTcp = server.addEndpoint( 
        RCF::TcpEndpoint(50001) );
    RCF::ServerTransport & serverTransportUdp = server.addEndpoint( 
        RCF::UdpEndpoint(50002) );
    server.start();
```
或者，如果RcfServer只有一个服务器传输，可以通过调用RCF::RcfServer::getServerTransport()来访问它：
```cpp
    RCF::RcfServer server( RCF::TcpEndpoint(50001) );
    RCF::ServerTransport & serverTransport = server.getServerTransport();
```
在客户端，客户端传输可以通过RCF::ClientStub::getTransport()：
```cpp
    RcfClient<I_Echo> client( RCF::TcpEndpoint(50001) );
    RCF::ClientTransport & clientTransport = client.getClientStub().getTransport();
```
传输配置
最大传入消息长度
对于服务器端传输，通常需要设置传入网络消息大小的上限。如果没有上限，格式不正确的请求可能导致服务器上任意大小的内存分配。

RCF服务器传输的最大传入消息长度设置默认为1 Mb，可以通过调用RCF::ServerTransport::setMaxIncomingMessageLength()来更改：
```cpp
    RCF::RcfServer server( RCF::TcpEndpoint(0) );
    // Set max message length to 5 Mb.
    server.getServerTransport().setMaxIncomingMessageLength(5*1024*1024);
```
类似地，客户机传输也有一个最大传入消息长度设置。它默认为1mb，可以通过调用RCF::ClientTransport::setMaxIncomingMessageLength()来更改：
```cpp
    RcfClient<I_Echo> client( RCF::TcpEndpoint(123) );
    // Set max message length to 5 Mb.
    client.getClientStub().getTransport().setMaxIncomingMessageLength(5*1024*1024);
```
由于客户端传输仅从其连接到的对等端接收网络消息，因此不正确数据包的风险不如服务器传输的大。

可以查询RcfClient<>查询发送的最新请求和响应消息的大小：
```cpp
        RcfClient<I_Echo> client( RCF::TcpEndpoint(50001) );
        client.Echo("1234");
        // Retrieve request and response size of the previous call.
        RCF::ClientTransport & transport = client.getClientStub().getTransport();
        std::size_t requestSize = transport.getLastRequestSize();
        std::size_t responseSize = transport.getLastResponseSize();
```
连接限制
要设置RCF服务器传输的最大同时连接数：
```cpp
    RCF::RcfServer server( RCF::TcpEndpoint(0) );
    // Allow at most 100 clients to be connected at any time.
    server.getServerTransport().setConnectionLimit(100);
```
此设置与UDP服务器传输无关，因为UDP协议中没有连接的概念。

系统端口选择
对于基于ip的服务器传输，您可以通过指定0作为端口号，允许本地系统自动分配服务器端口号。当服务器启动时，系统将找到一个空闲端口并将其分配给服务器。端口号可以通过RCF::IpServerTransport::getPort()：
```cpp
    RCF::RcfServer server( RCF::TcpEndpoint(0) );
    server.start();
    int port = server.getIpServerTransport().getPort();
    RcfClient<I_Echo> client(( RCF::TcpEndpoint(port) ));
```
基于ip的访问规则
对于基于IP的服务器传输，可以根据客户机的IP地址允许或拒绝客户机访问。

要配置允许客户机使用的IP规则，请使用RCF::IpServerTransport::setAllowIps()：
```cpp
    RCF::RcfServer server( RCF::TcpEndpoint(0) );
    RCF::IpServerTransport & ipTransport = 
        dynamic_cast<RCF::IpServerTransport &>(server.getServerTransport());
    std::vector<RCF::IpRule> rules;
    // Match 11.22.33.* (24 significant bits).
    rules.push_back( RCF::IpRule( RCF::IpAddress("11.22.33.0"), 24) );
    ipTransport.setAllowIps(rules);
    server.start();
    // Access will be granted to clients connecting from IP addresses matching 11.22.33.* .
    // All other clients will be denied.
```
要配置用于拒绝客户机的IP规则，请使用RCF::IpServerTransport::setDenyIps()：
```cpp
    RCF::RcfServer server( RCF::TcpEndpoint(0) );
    RCF::IpServerTransport & ipTransport = 
        dynamic_cast<RCF::IpServerTransport &>(server.getServerTransport());
    std::vector<RCF::IpRule> rules;
    // Match 11.*.*.* (8 significant bits).
    rules.push_back( RCF::IpRule( RCF::IpAddress("11.0.0.0"), 8) );
    // Match 12.22.*.* (16 significant bits).
    rules.push_back( RCF::IpRule( RCF::IpAddress("12.22.0.0"), 16) );
    ipTransport.setDenyIps(rules);
    server.start();
    // Access will be denied to clients connecting from IP addresses matching 11.*.*.*  and 12.22.*.* .
    // All other clients will be allowed.
```
IPv4 / IPv6
RCF同时支持IPv4和IPv6。默认情况下，在RCF中启用了IPv6支持，但是您可以定义RCF_FEATURE_IPV6=0来禁用它(参见构建RCF)。

例如，要在环回IPv4连接上运行服务器和客户机：
```cpp
        // Specifying an explicit IPv4 address to bind to.
        RCF::RcfServer server( RCF::TcpEndpoint("127.0.0.1", 50001) );
        server.start();
        // Specifying an explicit IPv4 address to bind to.
        RcfClient<I_Echo> client( RCF::TcpEndpoint("127.0.0.1", 50001) );
```
要在环回IPv6连接上运行服务器和客户机，请指定::1而不是127.0.0.1：
```cpp
        // Specifying an explicit IPv6 address to bind to.
        RCF::RcfServer server( RCF::TcpEndpoint("::1", 50001) );
        server.start();
        // Specifying an explicit IPv6 address to bind to.
        RcfClient<I_Echo> client( RCF::TcpEndpoint("::1", 50001) );
```
RCF使用POSIX getaddrinfo()函数来解析IP地址。getaddrinfo()可以返回IPv4或IPv6地址，这取决于本地系统和网络的配置。因此，下面的客户机将使用IPv4或IPv6，这取决于本地系统和网络是如何配置的：
```cpp
        // Will resolve to either IPv4 or IPv6, depending on what the system 
        // resolves machine.domain to.
        RcfClient<I_Echo> client( RCF::TcpEndpoint("machine.domain", 50001) );
```
您可以使用RCF::IpAddressV4和RCF::IpAddressV6类强制执行IPv4或IPv6解析：
```cpp
        // Force IPv4 address resolution.
        RCF::IpAddressV4 addr_4("machine.domain", 50001);
        RcfClient<I_Echo> client_4(( RCF::TcpEndpoint(addr_4) ));
        // Force IPv6 address resolution.
        RCF::IpAddressV6 addr_6("machine.domain", 50001);
        RcfClient<I_Echo> client_6(( RCF::TcpEndpoint(addr_6) ));
```
在具有双IPv4/IPv6堆栈的机器上，您可能希望您的服务器同时监听IPv4和IPv6地址。要实现可移植，您应该同时监听0.0.0.0和::0：
```cpp
        // Listen on port 50001, on both IPv4 and IPv6.
        RCF::RcfServer server;
        server.addEndpoint( RCF::TcpEndpoint("0.0.0.0", 50001) );
        server.addEndpoint( RCF::TcpEndpoint("::0", 50001) );
        server.start();
```
在一些平台上，只监听::0就足够了，因为系统将把传入的IPv4连接转换为IPv6连接，使用特殊的IPv6地址类。

为客户端绑定本地地址
当RcfClient<>使用基于ip的传输连接到服务器时，默认行为是允许系统决定使用哪个本地网络接口和本地端口。在某些情况下，您可能希望显式地设置客户机应该绑定到的本地网络接口。

您可以在连接之前调用RCF::IpClientTransport::setLocalIp()来实现这一点：
```cpp
        RcfClient<I_Echo> client( RCF::TcpEndpoint("127.0.0.1", 50001) );
        
        RCF::IpClientTransport & ipTransport = 
            client.getClientStub().getIpTransport();
        // Force client to bind to a particular local network interface (127.0.0.1).
        ipTransport.setLocalIp( RCF::IpAddress("127.0.0.1", 0) );
        client.getClientStub().connect();
```
连接好RcfClient<>之后，您可以通过调用RCF::IpClientTransport::getAssignedLocalIp()来确定它绑定到哪个本地网络接口和端口：
```cpp
        RcfClient<I_Echo> client( RCF::TcpEndpoint("127.0.0.1", 50001) );
        RCF::IpClientTransport & ipTransport = 
            client.getClientStub().getIpTransport();
        client.getClientStub().connect();
        // Find out which local network interface the client is bound to.
        RCF::IpAddress localIp = ipTransport.getAssignedLocalIp();
        std::string localInterface = localIp.getIp();
        int localPort = localIp.getPort();
```
套接字级访问
RCF提供对客户机和服务器传输的底层OS原语(如套接字和句柄)的访问。例如：
```cpp
        // Client-side.
        RcfClient<I_Echo> client( RCF::TcpEndpoint("127.0.0.1", 50001) );
        client.getClientStub().connect();
        RCF::TcpClientTransport & tcpClientTransport = 
            dynamic_cast<RCF::TcpClientTransport &>( 
                client.getClientStub().getTransport() );
        // Obtain client socket handle.
        int sock = tcpClientTransport.getNativeHandle();
```
```cpp
        // Server-side.
        RCF::NetworkSession & networkSession = RCF::getCurrentRcfSession().getNetworkSession();
        
        RCF::TcpNetworkSession & tcpNetworkSession = 
            dynamic_cast<RCF::TcpNetworkSession &>(networkSession);
        // Obtain server socket handle.
        int sock = tcpNetworkSession.getNativeHandle();
```
如果需要设置自定义套接字选项，这将非常有用。

传输的实现
本节介绍RCF中实现的各种传输类型。

TCP
TCP端点在RCF中由RCF::TcpEndpoint类表示，该类由IP地址和端口号构造。
```cpp
        RCF::RcfServer server( RCF::TcpEndpoint("0.0.0.0", 50001) );
        server.start();
        RcfClient<I_Echo> client( RCF::TcpEndpoint("127.0.0.1", 50001) );
```
服务器传输将IP地址解释为要监听的本地网络接口。例如，为了监听所有可用的IPv4网络接口，应该指定“0.0.0.0”，而“127.0.0.1”应该指定为只监听环回IPv4接口。如果没有指定IP地址，则假设“127.0.0.1”。

UDP
与RCF::TcpEndpoint类似，RCF::UdpEndpoint由IP地址和端口构造。
```cpp
        RCF::RcfServer server( RCF::UdpEndpoint("0.0.0.0", 50001) );
        server.start();
        RcfClient<I_Echo> client( RCF::UdpEndpoint("127.0.0.1", 50001) );
```
UdpEndpoint还包含一些额外的功能，用于处理多播和广播。

多播
UdpEndpoint可以配置为监听多播IP地址：
```cpp
        // Listen on multicast address 232.5.5.5, on port 50001, on all network interfaces.
        RCF::UdpEndpoint udpEndpoint("0.0.0.0", 50001);
        udpEndpoint.listenOnMulticast("232.5.5.5");
        RCF::RcfServer server(udpEndpoint);
        server.start();
```
注意，服务器仍然需要指定一个本地网络接口来监听。

要发送组播消息，在创建客户端时指定一个组播IP地址和端口：
```cpp
            RcfClient<I_Echo> client( RCF::UdpEndpoint("232.5.5.5", 50001) );
            client.Echo(RCF::Oneway, "ping");
```
广播
要发送广播消息，请指定广播IP地址和端口：
```cpp
            RcfClient<I_Echo> client( RCF::UdpEndpoint("255.255.255.255", 50001) );
            client.Echo(RCF::Oneway, "ping");
```
地址共享
RCF的UDP服务器传输可以配置为共享其地址绑定，这样多个RcfServer就可以在同一接口的同一端口上侦听。默认情况下，侦听多播地址时启用此功能，但也可以在侦听非多播地址时启用。如果同一台机器上的多个进程需要监听相同的广播，这将非常有用：
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
        // This broadcast message will be received by both servers.
        RcfClient<I_Echo> client( RCF::UdpEndpoint("255.255.255.255", 50001) );
        client.Echo(RCF::Oneway, "ping");
```
服务器发现
在服务器是在动态分配的端口上启动的情况下，多播和广播是向客户机通信服务器IP地址和端口的一种有用方法。例如：
```cpp
// Interface for broadcasting port number of a TCP server.
RCF_BEGIN(I_Broadcast, "I_Broadcast")
    RCF_METHOD_V1(void, ServerIsRunningOnPort, int)
RCF_END(I_Broadcast)
// Implementation class for receiving I_Broadcast messages.
class BroadcastImpl
{
public:
    BroadcastImpl() : mPort()
    {}
    void ServerIsRunningOnPort(int port)
    {
        mPort = port;
    }
    int mPort;
};
// A server thread runs this function, to broadcast the server location once 
// per second.
void broadcastThread(int port, const std::string &multicastIp, int multicastPort)
{
    RcfClient<I_Broadcast> client( 
        RCF::UdpEndpoint(multicastIp, multicastPort) );
    client.getClientStub().setRemoteCallMode(RCF::Oneway);
    // Broadcast 1 message per second.
    while (true)
    {
        client.ServerIsRunningOnPort(port);
        RCF::sleepMs(1000);
    }
}
```
```cpp
        // ***** Server side ****
        // Start a server on a dynamically assigned port.
        EchoImpl echoImpl;
        RCF::RcfServer server( RCF::TcpEndpoint(0));
        server.bind<I_Echo>(echoImpl);
        server.start();
        // Retrieve the port number.
        int port = server.getIpServerTransport().getPort();        
        // Start broadcasting the port number.
        RCF::ThreadPtr broadcastThreadPtr(new RCF::Thread(
            [=]() { broadcastThread(port, "232.5.5.5", 50001); }));
        // ***** Client side ****
        // Clients will listen for the broadcasts before doing anything else.
        
        RCF::UdpEndpoint udpEndpoint("0.0.0.0", 50001);
        udpEndpoint.listenOnMulticast("232.5.5.5");
        RCF::RcfServer clientSideBroadcastListener(udpEndpoint);
        BroadcastImpl broadcastImpl;
        clientSideBroadcastListener.bind<I_Broadcast>(broadcastImpl);
        clientSideBroadcastListener.start();
        // Wait for a broadcast message.
        while (!broadcastImpl.mPort)
        {
            RCF::sleepMs(1000);
        }
        // Once the clients know the port number, they can connect.
        RcfClient<I_Echo> client( RCF::TcpEndpoint(broadcastImpl.mPort));
        client.Echo("asdf");
```
注意，这里我们实际上使用多播地址向客户端广播信息。如果在这个特定的网络上无法进行多播，我们也可以使用IP广播地址而不是IP多播地址。

Win32命名管道
RCF支持Win32命名管道传输。RCF::Win32NamedPipeEndpoint接受一个构造函数参数，它是命名管道的名称，带或不带前导\\。管道\ \前缀。
```cpp
        RCF::RcfServer server( RCF::Win32NamedPipeEndpoint("PrintSvrPipe") );
        server.start();
        RcfClient<I_Echo> client( RCF::Win32NamedPipeEndpoint("PrintSvrPipe") );
```
使用Win32命名管道的一个优点是，它们允许对客户机进行简单的身份验证。使用Win32命名管道服务器传输的服务器可以通过RCF::Win32NamedPipeImpersonator类对其客户端进行身份验证，该类使用Windows API函数ImpersonateNamedPipeClient()来模拟客户端：
```cpp
RCF_BEGIN(I_Echo, "I_Echo")
    RCF_METHOD_R1(std::string, Echo, const std::string &)
RCF_END(I_Echo)
class EchoImpl
{
public:
    std::string Echo(const std::string & s)
    {
        // Impersonate client.
        RCF::Win32NamedPipeImpersonator impersonator(RCF::getCurrentRcfSession());
        std::cout << "Client user name: " << RCF::getMyUserName();
        return s;
    }
};
```
如果使用到127.0.0.1的TCP连接，则需要启用Kerberos或NTLM身份验证来安全地确定客户机用户名(请参阅传输协议)。

UNIX域套接字
UNIX域套接字的功能类似于Win32命名管道，允许在同一台机器上的服务器和客户机之间进行有效的通信。:UnixLocalEndpoint接受一个参数，它是UNIX域套接字的名称。名称必须是有效的文件系统路径。对于服务器，程序必须具有创建给定路径的足够权限，并且文件必须不存在。对于客户机，程序必须具有访问给定路径的足够权限。

下面是一个简单的例子：
```cpp
    RCF::RcfServer server( RCF::UnixLocalEndpoint("/home/xyz/MySocket"));
    server.start();
    RcfClient<I_Echo> client( RCF::UnixLocalEndpoint("/home/xyz/MySocket"));
```
HTTP / HTTPS
RCF支持通过HTTP和HTTPS协议进行隧道化远程调用。特别是，远程调用可以通过HTTP和HTTPS代理进行定向。

HTTPS本质上是在SSL协议之上分层的HTTP协议。因此，HTTPS的SSL方面的配置与SSL传输协议(请参阅传输协议)的配置方式相同。

服务器端
要使用HTTP端点设置服务器，请使用RCF::HttpEndpoint：
```cpp
        RCF::RcfServer server( RCF::HttpEndpoint("0.0.0.0", 80) );
        PrintService printService;
        server.bind<I_PrintService>(printService);
        server.start();
```
类似地，对于一个HTTPS endpoint, use RCF：
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
客户端
客户机端配置类似，使用RCF::HttpEndpoint作为HTTP客户机：
```cpp
        RcfClient<I_PrintService> client( RCF::HttpEndpoint("printsvr.acme.com", 80) );
        client.Print("Hello World");
```
， RCF::HttpsEndpoint for a HTTPS客户端：
```cpp
        RcfClient<I_PrintService> client( RCF::HttpsEndpoint("printsvr.acme.com", 443) );
        client.getClientStub().setCertificateValidationCallback(&schannelValidate);
        client.Print("Hello World");
```
最后，要通过HTTP或HTTPS代理直接进行远程调用，请使用RCF::ClientStub::setHttpProxy()和RCF::ClientStub::setHttpProxyPort()函数：
```cpp
            client.getClientStub().setHttpProxy("web-proxy.acme.com");
            client.getClientStub().setHttpProxyPort(8080);
            client.Print("Hello World");
```
反向代理
HTTP反向代理在Internet上很常见，用于为后端HTTP服务器提供负载平衡和SSL卸载等功能。

反向代理通常对客户机和目标服务器是透明的。您可以在RCF客户机和服务器之间放置反向代理，这为跨一组RCF服务器分配负载提供了开箱即用的机制。

在负载平衡场景中，您可能希望RCF客户机继续向它最初连接到的后端服务器发送请求。为此，您需要使用支持会话关联的反向代理。会话关联通常由反向代理实现，方法是在通信流的初始HTTP响应中插入一个特殊的HTTP cookie。然后，RCF客户机将在后续请求中自动包含此cookie，从而允许反向代理将后续请求路由到相同的后端服务器。

如果您有多个RCF客户机连接，您可以将它们配置为连接到相同的后端服务器，方法是从第一个连接检索HTTP cookie，然后将其应用于其他连接(请参阅RCF::ClientStub::setHttpCookies())。

使用RCF可以使用的另一个反向代理特性是SSL卸载。通过卸载SSL, RCF客户机使用HTTPS连接进行连接，但是HTTPS连接由反向代理解密，解密后的HTTP流被转发到后端服务器。这允许客户端和后端服务器之间进行安全通信，而反向代理承担SSL加密/解密的负载。这还具有将证书配置从多个后端服务器集中到反向代理服务器的优势。

RCF已通过以下反向代理进行测试：
IIS ARR
nginx