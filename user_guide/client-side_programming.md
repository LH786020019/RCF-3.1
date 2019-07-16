<!--
 * @Author: haoluo
 * @Date: 2019-07-15 11:44:30
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-16 16:00:09
 * @Description: file content
 -->
## 客户端编程(Client-side Programming)
### 1. 进行远程调用
远程调用总是通过一个 `RcfClient<>` 实例进行的。一旦您定义了一个 RCF 接口：
```cpp
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
```
，然后您就可以实例化 `RcfClient<>` 并在其上调用一个方法：
```cpp
RcfClient<I_PrintService> client((RCF::TcpEndpoint(port)));
client.Print("Hello World");
```
一个 `RcfClient<>` 控制到 server 的单个网络连接，并且一次只能由一个线程使用。当进行第一次远程调用时，它将使用传递给 RcfClient<> 构造函数的端点参数自动连接到 server 。当 RcfClient<> 被销毁时，与 server 的网络连接将被关闭。

### 2. 客户端存根(Client Stubs)
远程调用的客户端是通过 [RCF::ClientStub](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html) 控制的。每个 `RcfClient<>` 实例都包含一个 `RCF::ClientStub`，可以通过调用 `getClientStub()` 访问它。
```cpp
RCF::ClientStub & clientStub = client.getClientStub();
```
几乎所有远程调用的客户端配置都是通过 RCF::ClientStub 完成的。

### 3. 传输访问
您可以使用 [RCF::ClientStub::getTransport()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#aed25109a12fb5eff358971769a9a772d) 访问一个 ClientStub 的底层传输。
```cpp
RCF::ClientTransport & transport = client.getClientStub().getTransport();
```
您可以配置许多特定于传输的设置（ 请参阅 [RCF::ClientTransport](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_transport.html) ）。

从一个 server 传输级连接和断开连接通常是自动处理的。但是，您也可以通过调用 [RCF::ClientStub::connect()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a3f4b0d9af03e87dee462aa24fbe66415) 和 [RCF::ClientStub::disconnect()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a0872b5ad13e311c6e220777e3538390f) 手动连接和断开连接。
```cpp
// Connect to server.
client.getClientStub().connect();
// Disconnect from server.
client.getClientStub().disconnect();
```
您可以使用 `ClientStub::releaseTransport()` 和 `ClientStub::setTransport()` 将一个传输从一个 client stub 移动到另一个 client stub：
```cpp
RcfClient<I_AnotherInterface> client2( client.getClientStub().releaseTransport() );
client2.AnotherPrint("Hello World");
client.getClientStub().setTransport( client2.getClientStub().releaseTransport() );
client.Print("Hello World");
```
如果您希望使用相同的连接对不同的接口进行调用，这是非常有用的。

### 4. 超时(Timeouts)
有两个重要的超时设置会影响远程调用的执行方式。

第一个是连接超时，通过 [RCF::ClientStub::setConnectTimeoutMs()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a6c17010d3692c26f7c6c0c003898a373) 配置。此设置确定 client 在尝试建立到 server 的网络连接时将等待多长时间。您可以通过调用 [RCF::ClientStub::setConnectTimeoutMs()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a6c17010d3692c26f7c6c0c003898a373) 来为特定的 `RcfClient<>` 设置自定义连接超时，也可以通过调用 [RCF::globals()](http://www.deltavsoft.com/doc/group___functions.html#gafaaa3e03114ef72ab45876a4bb155b4d).setDefaultConnectTimeoutMs() 来为所有的 `RcfClient<>` 实例设置默认连接超时。

第二个是远程调用超时，通过 [RCF::ClientStub::setRemoteCallTimeoutMs()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#af913a97d88fed06930810c959c47deac) 配置。此设置确定客户端连接后等待一次远程调用完成的时间。与连接超时一样，您可以通过调用 [RCF::ClientStub::setRemoteCallTimeoutMs()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#af913a97d88fed06930810c959c47deac) 来为特定的 `RcfClient<>` 设置远程调用超时，也可以通过调用 [RCF::globals()](http://www.deltavsoft.com/doc/group___functions.html#gafaaa3e03114ef72ab45876a4bb155b4d).setDefaultRemoteCallTimeoutMs() 为所有 `RcfClient<>` 实例设置默认远程调用超时。

两个超时都以毫秒为单位指定。

### 5. Pinging
可以使用 `ClientStub::ping()` 方法来确定 client 是否具有到 server 的一个可行连接。`ping` 的行为与没有输入和输出参数的远程调用完全一样，并且受到相同的超时限制。
```cpp
// Ping the server.
client.getClientStub().ping();
```

### 6. 客户机进度通知(Client Progress Notification)
默认情况下，远程调用是同步执行的。这意味着在执行一次远程调用时，client 线程被阻塞。不过，您可以将 client 配置为定期向一个自定义 client 进度回调函数报告进度。通过使用进度回调函数，您可以执行应用程序特定的代码，例如显示一个进度条、刷新应用程序 GUI 或取消调用。
```cpp
std::uint32_t progressCallbackIntervalMs = 500;
auto progressCallback = [&](const RCF::RemoteCallProgressInfo& info, RCF::RemoteCallAction& action) { 
    // 设置为 RCF::Rca_Cancel 以取消调用
    action = RCF::Rca_Continue;
};
client.getClientStub().setRemoteCallProgressCallback(
    progressCallback,
    progressCallbackIntervalMs);
// 在调用执行时，将每 500ms 调用一次 progressCallback
client.Print("Hello World");
```
异步远程调用（ 请参阅[异步调用](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/asynchronous_remote_calls.md) ），是在一次远程调用期间重新获得控制权的另一种方法。

### 7. 远程调用模式
每个 RCF client 都有一个与之关联的远程调用模式。有两种基本的远程调用模式：
- 双向调用( `RCF::Twoway` )直到 client 收到来自 server 的响应才会完成。双向调用是默认的远程调用模式。
- 单向调用( `RCF::Oneway` )一旦请求被发送到 server 就会完成。Server 不向单向调用发送任何响应。

您可以使用 [RCF::ClientStub::setRemoteCallMode()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a76a6eca176ed3a10ae669993c130271c) 来设置远程调用模式。
```cpp
client.getClientStub().setRemoteCallMode(RCF::Oneway);
client.Print("Hello World");
client.getClientStub().setRemoteCallMode(RCF::Twoway);
client.Print("Hello World");
```
远程调用模式也可以指定为远程调用的一部分，作为远程调用的第一个参数：
```cpp
client.Print(RCF::Oneway, "Hello World");
client.Print(RCF::Twoway, "Hello World");
```
在本例中，作为调用的一部分指定的远程调用模式覆盖 [RCF::ClientStub::setRemoteCallMode()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a76a6eca176ed3a10ae669993c130271c) 设置的模式。

### 8. 批量单向调用(Batched One-way Calls)
RCF 支持批量单向调用，作为单向调用的一个扩展。

当一个 RcfClient<> 被配置为进行单向调用时，将为所进行的每个远程调用向 server 发送一条网络消息。如果正在进行大量调用，可以使用批量单向调用将多个单向消息合并到单个网络消息中，同时仍然在 server 上按顺序执行远程调用。

根据发送的消息的数量和频率，这可能导致显著的性能提高。

要配置批量单向调用，可以使用 [RCF::ClientStub::enableBatching()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a1cdcb0d45fdf04d1193e9e563cdfb835)、[RCF::ClientStub::disableBatching()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#aed7df01dae6ebe70eef0b902fc0cfb10) 和 [RCF::ClientStub::flushBatch()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a7d779b6c644111a9f4370f657bc22fff) 函数：
```cpp
client.getClientStub().enableBatching();
// 当消息大小接近 50kb 时自动发送批处理。
client.getClientStub().setMaxBatchMessageLength(1024*50);
for (std::size_t i=0; i<100; ++i){
    client.Print("Hello World");
}
// Send final batch.
client.getClientStub().flushBatch();
```
### 9. 自定义请求和响应数据
通常，一次远程调用中从 client 传输到 server 的应用程序数据包含在远程调用的参数中。您还可以通过设置远程调用请求的用户数据字段来传输额外的信息。然后就可以在 server 上访问这些数据以及远程调用的参数。

类似地，server 代码可以设置远程调用响应的用户数据字段，并将该数据连同远程调用的参数一起传输回 client。

用户数据必须指定为一个字符串。
```cpp
class PrintService{
public:
    void Print(const std::string & s){
        std::cout << "I_PrintService service: " << s << std::endl;
        RCF::RcfSession & session = RCF::getCurrentRcfSession();
        
        std::string customRequestData = session.getRequestUserData();
        std::cout << "Custom request data: " << customRequestData << std::endl;
        std::string customResponseData = "f81d4fae-7dec-11d0-a765-00a0c91e6bf6";
        session.setResponseUserData(customResponseData);
    }
};
```
```cpp
    client.getClientStub().setRequestUserData( "e6a9bb54-da25-102b-9a03-2db401e887ec" );
    client.Print("Hello World");
    std::string customReponseData = client.getClientStub().getResponseUserData();
    std::cout << "Custom response data: " << customReponseData << std::endl;
```
用户数据字段可用于实现自定义身份验证方案，其中需要将一个 `token(令牌)` 作为每个远程调用的一部分传递，但您不希望令牌成为远程调用接口的一部分。

### 10. 复制语义(Copy semantics)
可以复制 RcfClient<> 对象。但是，一个 RcfClient<> 对象的每个副本都将建立到 server 的自己的网络连接。下面的代码将建立到 server 的 3 个网络连接：
```cpp
RcfClient<I_PrintService> client1(( RCF::TcpEndpoint(port) ));
RcfClient<I_PrintService> client2(client1);
RcfClient<I_PrintService> client3;
client3 = client1;
client1.Print("Hello World");
client2.Print("Hello World");
client3.Print("Hello World");
```