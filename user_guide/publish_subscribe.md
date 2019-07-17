<!--
 * @Author: haoluo
 * @Date: 2019-07-16 10:10:05
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-17 18:44:00
 * @Description: file content
 -->
## 发布/订阅(Publish/Subscribe)
RCF 提供了一个内置的发布/订阅实现。

发布/订阅是一种消息传递范例，发布者(publisher)将消息发送给订阅者(subscriber)组。发布者不知道单个订阅者并不能直接向他们发送消息，而是将他们的消息分类为主题(topic)，订阅者选择从哪个主题接收消息。当发布者发布关于特定主题的消息时，订阅该主题的所有订阅者都将收到该消息。

发布/订阅可以在防火墙和 NAT 存在的情况下使用。唯一的网络拓扑要求是订阅者必须能够发起到发布者的网络连接。发布者永远不会尝试与它们的订阅者建立网络连接。

### 1. 发布者(Publishers)
要创建一个发布者，请使用 [RCF::RcfServer::createPublisher<>()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#a3897d6a968fc0d9c4c04ae2d3c1911f8)：
```cpp
    RCF::RcfServer pubServer( RCF::TcpEndpoint(50001) );
    pubServer.start();
    typedef RCF::Publisher<I_PrintService> PrintServicePublisher;
    typedef std::shared_ptr< PrintServicePublisher > PrintServicePublisherPtr;
    PrintServicePublisherPtr publisherPtr = pubServer.createPublisher<I_PrintService>();
```
这将创建一个具有默认主题名称的发布者。默认的主题名称是传递给 [RCF::RcfServer::createPublisher<>()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#a3897d6a968fc0d9c4c04ae2d3c1911f8) 的 RCF 接口的运行时名称(在本例中是 `"I_PrintService"`)。

还可以显式设置主题名称。例如，使用相同的 RCF 接口创建两个主题名称不同的发布者：
```cpp
    RCF::PublisherParms pubParms;
    pubParms.setTopicName("PrintService_Topic_1");
    PrintServicePublisherPtr publisher1Ptr = pubServer.createPublisher<I_PrintService>(pubParms);
    pubParms.setTopicName("PrintService_Topic_2");
    PrintServicePublisherPtr publisher2Ptr = pubServer.createPublisher<I_PrintService>(pubParms);
```
[RCF::Publisher<>](http://www.deltavsoft.com/doc/class_r_c_f_1_1_publisher.html) 对象由 [RCF::RcfServer::createPublisher<>()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#a3897d6a968fc0d9c4c04ae2d3c1911f8) 返回，用于发布远程调用。已发布的远程调用始终具有单向语义，并且由当前订阅该发布主题的所有订阅者接收。要发布一个调用，请使用 `RCF::Publisher<>::publish()`：
```cpp
    publisherPtr->publish().Print("First published message.");
    publisherPtr->publish().Print("Second published message.");
```
当 [RCF::Publisher<>](http://www.deltavsoft.com/doc/class_r_c_f_1_1_publisher.html) 对象被销毁时，或者当调用 `RCF::Publisher<>::close()` 时，发布主题被关闭，所有订阅者断开连接。
```cpp
    // 关闭发布主题。所有订阅者将断开连接。
    publisherPtr->close();
```

### 2. 订阅者(Subscribers)
要订阅一个发布者，请使用 [RCF::RcfServer::createSubscription<>()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#af8e3e5d679e47140e1560fa2915252d1)：
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
对于发布者，主题名称默认为 RCF 接口的运行时名称(在本例中为 `"I_PrintService"`)。主题名称也可以显式指定：
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
传递给 [RCF::RcfServer::createSubscription<>()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#af8e3e5d679e47140e1560fa2915252d1) 的第一个参数是接收已发布消息的对象。应用程序的职责是确保在订阅仍然连接时不销毁此对象。

可以提供一个断开连接的回调函数，如果订阅者被断开连接，该函数将被调用：
```cpp
void onSubscriptionDisconnected(RCF::RcfSession & session){
    // 在这里处理订阅断开。
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
若要终止一个订阅、销毁 `Subscription` 对象或调用 `Subscription::close()`：
```cpp
        subscriptionPtr->close();
```

### 3. 访问控制
访问控制可以以一个访问控制回调函数的形式应用于发布者，该回调函数将在发布 server 上为每个试图设置订阅的订阅者调用。与服务绑定访问控制类似，发布者访问控制可用于检查订阅者连接的 [RCF::RcfSession](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_session.html)，以获取任何相关的身份验证信息：
```cpp
bool onSubscriberConnect(RCF::RcfSession & session, const std::string & topicName){
    // 返回 true 以允许访问，否则返回 false。
    // ...
    return true;
}
void onSubscriberDisconnect(RCF::RcfSession & session, const std::string & topicName){
    // ...
}
```
```cpp
        RCF::PublisherParms pubParms;
        pubParms.setOnSubscriberConnect(onSubscriberConnect);
        pubParms.setOnSubscriberDisconnect(onSubscriberDisconnect);
        PrintServicePublisherPtr publisher1Ptr = pubServer.createPublisher<I_PrintService>(pubParms);
```