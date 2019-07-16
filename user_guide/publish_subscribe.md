<!--
 * @Author: haoluo
 * @Date: 2019-07-16 10:10:05
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-16 10:13:06
 * @Description: file content
 -->
## Publish/Subscribe
RCF提供了一个内置的发布/订阅实现。

发布/订阅是一种消息传递范例，发布者将消息发送给订阅用户组。发布者不知道单个订阅者并直接向他们发送消息，而是将他们的消息分类为主题，订阅者选择从哪个主题接收消息。当发布者发布关于特定主题的消息时，订阅该主题的所有订阅者都将收到该消息。

发布/订阅可以在防火墙和NAT存在的情况下使用。惟一的网络拓扑要求是订阅者必须能够发起到发布服务器的网络连接。发布者永远不会尝试与订阅者建立网络连接。

出版商
要创建发布者，请使用RCF::RcfServer::createPublisher<>()：
```cpp
    RCF::RcfServer pubServer( RCF::TcpEndpoint(50001) );
    pubServer.start();
    typedef RCF::Publisher<I_PrintService> PrintServicePublisher;
    typedef std::shared_ptr< PrintServicePublisher > PrintServicePublisherPtr;
    PrintServicePublisherPtr publisherPtr = pubServer.createPublisher<I_PrintService>();
```
这将创建一个具有默认主题名称的发布者。默认的主题名称是传递给RCF::RcfServer::createPublisher<>()的RCF接口的运行时名称(在本例中是“I_PrintService”)。

还可以显式设置主题名称。例如，使用相同的RCF接口创建两个主题名称不同的发布者：
```cpp
    RCF::PublisherParms pubParms;
    pubParms.setTopicName("PrintService_Topic_1");
    PrintServicePublisherPtr publisher1Ptr = pubServer.createPublisher<I_PrintService>(pubParms);
    pubParms.setTopicName("PrintService_Topic_2");
    PrintServicePublisherPtr publisher2Ptr = pubServer.createPublisher<I_PrintService>(pubParms);
```
RCF::Publisher<>对象由RCF::RcfServer::createPublisher<>()返回，用于发布远程调用。已发布的远程调用始终具有单向语义，并且由当前订阅该发布主题的所有订阅者接收。要发布调用，请使用RCF::Publisher<>::publish()：
```cpp
    publisherPtr->publish().Print("First published message.");
    publisherPtr->publish().Print("Second published message.");
```
当RCF::Publisher<>对象被销毁时，或者当调用RCF::Publisher<>::close()时，发布主题被关闭，所有订阅者断开连接。
```cpp
    // Close the publishing topic. All subscribers will be disconnected.
    publisherPtr->close();
```
用户
要订阅发布者，请使用RCF::RcfServer:: createsub< >()：
```cpp
    RCF::RcfServer subServer( RCF::TcpEndpoint(-1) );
    subServer.start();
    PrintService printService;
    RCF::SubscriptionParms subParms;
    subParms.setPublisherEndpoint( RCF::TcpEndpoint(50001) );
    RCF::SubscriptionPtr subscriptionPtr = subServer.createSubscription<I_PrintService>(
        printService, 
        subParms);
```
对于发布者，主题名称默认为RCF接口的运行时名称(在本例中为“I_PrintService”)。主题名称也可以显式指定：
```cpp
        PrintService printService;
        RCF::SubscriptionParms subParms;
        subParms.setPublisherEndpoint( RCF::TcpEndpoint(50001) );
        subParms.setTopicName("PrintService_Topic1");
        RCF::SubscriptionPtr subscription1Ptr = subServer.createSubscription<I_PrintService>(
            printService, 
            subParms);
        subParms.setTopicName("PrintService_Topic2");
        RCF::SubscriptionPtr subscription2Ptr = subServer.createSubscription<I_PrintService>(
            printService,
            subParms);
```
传递给RCF::RcfServer:: createsub< >()的第一个参数是接收已发布消息的对象。应用程序的职责是确保在订阅仍然连接时不销毁此对象。

可以提供一个断开连接的回调函数，如果订阅者断开连接，该函数将被调用：
```cpp
void onSubscriptionDisconnected(RCF::RcfSession & session)
{
    // Handle subscription disconnection here.
    // ...
}
```
```cpp
        PrintService printService;
        RCF::SubscriptionParms subParms;
        subParms.setPublisherEndpoint( RCF::TcpEndpoint(50001) );
        subParms.setOnSubscriptionDisconnect(&onSubscriptionDisconnected);
        RCF::SubscriptionPtr subscriptionPtr = subServer.createSubscription<I_PrintService>(
            printService, 
            subParms);
```
若要终止订阅、销毁订阅对象或调用订阅::close()：
```cpp
        subscriptionPtr->close();
```
访问控制
访问控制可以以访问控制回调的形式应用于发布者，该回调将在发布服务器上为每个试图设置订阅的订阅者调用。与服务绑定访问控制类似，发布者访问控制可用于检查订阅者连接的RCF::RcfSession，以获取任何相关的身份验证信息：
```cpp
bool onSubscriberConnect(RCF::RcfSession & session, const std::string & topicName)
{
    // Return true to allow access, false otherwise.
    // ...
    return true;
}
void onSubscriberDisconnect(RCF::RcfSession & session, const std::string & topicName)
{
    // ...
}
```
```cpp
        RCF::PublisherParms pubParms;
        pubParms.setOnSubscriberConnect(onSubscriberConnect);
        pubParms.setOnSubscriberDisconnect(onSubscriberDisconnect);
        PrintServicePublisherPtr publisher1Ptr = pubServer.createPublisher<I_PrintService>(pubParms);
```