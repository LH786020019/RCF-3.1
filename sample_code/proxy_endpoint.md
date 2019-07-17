<!--
 * @Author: haoluo
 * @Date: 2019-07-17 19:37:45
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-17 19:39:47
 * @Description: file content
 -->
## 代理端点
### 代理端点 Server
这个示例演示了一个公开代理端点的 server，允许 client 连接到目标 server。
```cpp
#include <iostream>
#include <RCF/RCF.hpp>
#include <RCF/ProxyEndpoint.hpp>
// Define RCF interface.
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
int main()
{
    try
    {
        RCF::RcfInit rcfInit;
        // Configure proxy endpoint server.
        RCF::RcfServer server(RCF::TcpEndpoint("127.0.0.1", 50001));
        server.setEnableProxyEndpoints(true);
        server.setThreadPool(RCF::ThreadPoolPtr(new RCF::ThreadPool(1, 10)));
        server.start();
        // Wait for destination server to come on line.
        std::string printServerName = "RoamingPrintSvr";
        bool done = false;
        while ( !done )
        {
            std::vector<std::string> proxyEndpoints;
            server.enumerateProxyEndpoints(proxyEndpoints);
            for ( auto proxyEndpoint : proxyEndpoints )
            {
                if ( proxyEndpoint == printServerName )
                {
                    {
                        std::cout << "In-process connection to proxy endpoint '" << printServerName << "'.";
                        RcfClient<I_PrintService> client(RCF::ProxyEndpoint(server, printServerName));
                        client.Print("Calling I_PrintService through a proxy endpoint");
                    }
                    {
                        std::cout << "Out-of-process connection to proxy endpoint '" << printServerName << "'.";
                        RcfClient<I_PrintService> client(RCF::ProxyEndpoint(RCF::TcpEndpoint("127.0.0.1", 50001), printServerName));
                        client.Print("Calling I_PrintService through a proxy endpoint");
                    }
                    done = true;
                }
            }
        }
    }
    catch ( const RCF::Exception & e )
    {
        std::cout << "Error: " << e.getErrorMessage() << std::endl;
    }
    return 0;
}
```

### 目标 Server
这个示例演示了通过代理端点接收调用的目标 server。
```cpp
#include <iostream>
#include <RCF/RCF.hpp>
#include <RCF/ProxyEndpoint.hpp>
// Define RCF interface.
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
// Server implementation of the I_PrintService RCF interface.
class PrintService
{
public:
    void Print(const std::string & s)
    {
        std::cout << "I_PrintService service: " << s << std::endl;
    }
};
int main()
{
    try
    {
        RCF::RcfInit rcfInit;
        // Configure destination server, using the network address of the proxy endpoint server.
        std::string printServerName = "RoamingPrintSvr";
        RCF::RcfServer server( RCF::ProxyEndpoint( RCF::TcpEndpoint("127.0.0.1", 50001), printServerName) );
        PrintService printService;
        server.bind<I_PrintService>(printService);
        server.start();
        // Destination server can now receive calls through the proxy endpoint server.
        // ...
        std::cout << "Press Enter to exit..." << std::endl;
        std::cin.get();
    }
    catch ( const RCF::Exception & e )
    {
        std::cout << "Error: " << e.getErrorMessage() << std::endl;
    }
    return 0;
}
```