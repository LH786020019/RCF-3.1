<!--
 * @Author: haoluo
 * @Date: 2019-07-17 19:34:42
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-17 19:36:34
 * @Description: file content
 -->
## Publish/subscribe
### 1. Publish/subscribe - Publisher
这个示例演示了一个 server 在 RCF 接口上发布调用。
```cpp
#include <iostream>
#include <RCF/RCF.hpp>
// Define RCF interface.
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
int main()
{
    try
    {
        RCF::RcfInit rcfInit;
        RCF::RcfServer server(RCF::TcpEndpoint("127.0.0.1", 50001));
        server.start();
        // Configure a publisher.
        typedef RCF::Publisher<I_PrintService>              PrintServicePublisher;
        typedef std::shared_ptr< PrintServicePublisher >    PrintServicePublisherPtr;
        PrintServicePublisherPtr publisherPtr = server.createPublisher<I_PrintService>();
        // Publish a message once every second.
        while ( true )
        {
            publisherPtr->publish().Print("Hello World");
            RCF::sleepMs(1000);
        }
    }
    catch ( const RCF::Exception & e )
    {
        std::cout << "Error: " << e.getErrorMessage() << std::endl;
    }
    return 0;
}
```
### 2. Publish/subscribe - Subscriber
这个示例演示了一个 server 接收来自发布者的调用。
```cpp
#include <iostream>
#include <RCF/RCF.hpp>
// Define RCF interface.
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
// This class will receive the published messages.
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
        
        PrintService printService;
        RCF::RcfServer server(RCF::TcpEndpoint(-1));
        server.start();
        // Configure a subscription.
        RCF::SubscriptionParms subParms;
        subParms.setPublisherEndpoint(RCF::TcpEndpoint("127.0.0.1", 50001));
        RCF::SubscriptionPtr subscriptionPtr = server.createSubscription<I_PrintService>(
            printService,
            subParms);
        // At this point, should be receiving published messages from the publisher.
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