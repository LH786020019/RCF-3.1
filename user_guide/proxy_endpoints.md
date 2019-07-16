<!--
 * @Author: haoluo
 * @Date: 2019-07-16 10:13:52
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-16 10:15:36
 * @Description: file content
 -->
## 代理端点
代理端点允许RCF客户机连接到它们无法直接连接的RCF服务器。

在许多情况下，客户机可能无法建立到它打算调用的服务器的直接网络连接。

客户端可能不知道服务器的网络地址。
服务器的网络地址可以是动态的。
可能存在防火墙或其他网络基础设施，阻止客户机到服务器的直接网络连接。
代理端点为这些情况提供了一个解决方案，允许一个RCF服务器(代理服务器)作为另一个RCF服务器(目标服务器)的代理。代理服务器维护来自目标服务器的连接池，并使用这些连接允许客户端连接到目标服务器。

配置代理服务器
代理服务器接受来自目标服务器的连接，并将这些连接保存在连接池中。然后，这些连接将用于希望连接到特定目标服务器的客户机。

每个目标服务器必须向代理服务器提供一个名称。目标服务器名称是任意的，由应用程序选择。客户机将使用目标服务器名称来指定要连接到哪个服务器。

要设置代理服务器，您可以像往常一样配置RCF::RcfServer，然后调用RCF::RcfServer:: setenableproxyendpoint()。一旦启动，RcfServer将开始注册目标服务器，并代理从客户机到这些目标服务器的连接。
```cpp
    // Proxy server.
    int proxyPort = 50001;
    RCF::RcfServer proxyServer( RCF::TcpEndpoint("0.0.0.0", proxyPort) );
    proxyServer.setEnableProxyEndpoints(true);
    proxyServer.setThreadPool(RCF::ThreadPoolPtr(new RCF::ThreadPool(1,10)));
    proxyServer.start();
```
您可以随时查询代理服务器，查询当前可用的代理端点的名称：
```cpp
    std::vector<std::string> proxyEndpointNames;
    proxyServer.enumerateProxyEndpoints(proxyEndpointNames);
```
配置目标服务器
目标RcfServer配置了一个RCF::ProxyEndpoint参数，指定代理服务器的网络地址，目标服务器的名称将公开为。
```cpp
    // Destination server.
    RCF::RcfServer destinationServer( RCF::ProxyEndpoint(RCF::TcpEndpoint(proxyIp, proxyPort), "RoamingPrintSvr") );
    PrintService printService;
    destinationServer.bind<I_PrintService>(printService);
    destinationServer.start();
```
一旦启动，目标RcfServer将开始启动到代理服务器的网络连接。客户端随后将使用这些连接对目标服务器进行远程调用。

通过代理服务器连接
一旦目标服务器和代理服务器启动并运行，客户机就可以使用RCF::ProxyEndpoint连接到目标服务器，并指定代理服务器和目标服务器的名称。
```cpp
    // Client calling through to proxy server, using in-process access to the proxy server.
    RcfClient<I_PrintService> client( RCF::ProxyEndpoint(proxyServer, "RoamingPrintSvr") );
    client.Print("Calling I_PrintService through a proxy endpoint.");
```
```cpp
    // Client calling through to proxy server, using network access to the proxy server.
    RcfClient<I_PrintService> client(RCF::ProxyEndpoint(RCF::TcpEndpoint(proxyIp, proxyPort), "RoamingPrintSvr"));
    client.Print("Calling I_PrintService through a proxy endpoint.");
```
使用代理连接，客户端可以像往常一样进行远程调用，并且能够断开连接并重新连接，就像使用未代理连接一样。不同的RCF客户机将具有到目标服务器的不同网络连接，在目标服务器上创建的任何会话对象的生存期都将映射到代理网络连接的客户机的生存期。