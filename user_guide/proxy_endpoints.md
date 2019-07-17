<!--
 * @Author: haoluo
 * @Date: 2019-07-16 10:13:52
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-17 18:43:51
 * @Description: file content
 -->
## 代理端点(Proxy Endpoints)
代理端点允许 RCF client 连接到它们无法直接连接的 RCF server。

在许多情况下，client 可能无法建立到它打算调用的 server 的直接网络连接。
- Client 可能不知道 server 的网络地址。
- Server 的网络地址可能是动态的。
- 可能存在防火墙或其他网络基础设施，阻止 client 到 server 的直接网络连接。

代理端点为这些情况提供了一个解决方案，允许一个 RCF server(代理 server) 作为另一个 RCF server(目标 server) 的代理。代理 server 维护来自目标 server 的连接池，并使用这些连接允许 client 连接到目标 server。

### 1. 配置一个代理 Server 
代理 server 接受来自目标 server 的连接，并将这些连接保存在连接池中。然后，这些连接将用于希望连接到特定目标 server 的 client。

每个目标 server 必须向代理 server 提供一个名称。目标 server 名称是任意的，由应用程序选择。Client 将使用目标 server 名称来指定要连接到哪个 server。

要设置代理 server，您可以像往常一样配置一个 [RCF::RcfServer](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html)，然后调用 [RCF::RcfServer::setEnableProxyEndpoints()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#a54730b50ed5e932c4ca237f59981d287)。一旦启动，`RcfServer` 将开始注册目标 server ，并代理从 client 到这些目标 server 的连接。
```cpp
    // Proxy server.
    int proxyPort = 50001;
    RCF::RcfServer proxyServer( RCF::TcpEndpoint("0.0.0.0", proxyPort) );
    proxyServer.setEnableProxyEndpoints(true);
    proxyServer.setThreadPool(RCF::ThreadPoolPtr(new RCF::ThreadPool(1,10)));
    proxyServer.start();
```
您可以随时查询代理 server，查询当前可用的代理端点的名称：
```cpp
    std::vector<std::string> proxyEndpointNames;
    proxyServer.enumerateProxyEndpoints(proxyEndpointNames);
```

### 2. 配置一个目标 Server 
目标 `RcfServer` 配置了一个 [RCF::ProxyEndpoint](http://www.deltavsoft.com/doc/class_r_c_f_1_1_proxy_endpoint.html) 参数，指定代理 server 的网络地址，目标 server 将公开为的名称。
```cpp
    // Destination server.
    RCF::RcfServer destinationServer( RCF::ProxyEndpoint(RCF::TcpEndpoint(proxyIp, proxyPort), "RoamingPrintSvr") );
    PrintService printService;
    destinationServer.bind<I_PrintService>(printService);
    destinationServer.start();
```
一旦启动，目标 `RcfServer` 将开始启动到代理 server 的网络连接。Client 随后将使用这些连接对目标 server 进行远程调用。

### 3. 通过一个代理 Server 连接
一旦目标 server 和代理 server 启动并运行，client 就可以使用 [RCF::ProxyEndpoint](http://www.deltavsoft.com/doc/class_r_c_f_1_1_proxy_endpoint.html) 连接到目标 server，并指定代理 server 和目标 server 的名称。
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
使用一个代理连接，client 可以像往常一样进行远程调用，并且能够断开连接并重新连接，就像使用未代理连接一样。不同的 RCF client 将具有到目标 server 的不同网络连接，在目标 server 上创建的任何会话对象的生命周期都将映射到代理网络连接的 client 的生命周期。