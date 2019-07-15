<!--
 * @Author: haoluo
 * @Date: 2019-07-15 11:44:30
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-15 11:48:31
 * @Description: file content
 -->
客户端编程
使远程调用
远程调用总是通过RcfClient<>实例进行的。一旦你定义了一个RCF接口：
```cpp
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
```
，然后可以实例化RcfClient<>并在其上调用一个方法：
```cpp
RcfClient<I_PrintService> client((RCF::TcpEndpoint(port)));
client.Print("Hello World");
```
RcfClient<>控制到服务器的单个网络连接，并且一次只能由一个线程使用。当进行第一次远程调用时，它将使用传递给RcfClient<>构造函数的端点参数自动连接到服务器。当RcfClient<>被销毁时，与服务器的网络连接将关闭。

客户端存根
远程调用的客户端是通过RCF::ClientStub控制的。每个RcfClient<>实例都包含一个RCF::ClientStub，可以通过调用getClientStub()访问它。
```cpp
RCF::ClientStub & clientStub = client.getClientStub();
```
几乎所有远程调用的客户端配置都是通过RCF::ClientStub完成的。

传输访问
可以使用RCF::ClientStub::getTransport()访问ClientStub的底层传输。
```cpp
RCF::ClientTransport & transport = client.getClientStub().getTransport();
```
您可以配置许多特定于传输的设置(请参阅RCF::ClientTransport)。

传输级连接和服务器断开连接通常是自动处理的。但是，您也可以通过调用RCF::ClientStub::connect()和RCF::ClientStub::disconnect()手动连接和断开连接。
```cpp
// Connect to server.
client.getClientStub().connect();
// Disconnect from server.
client.getClientStub().disconnect();
```
您可以使用ClientStub::releaseTransport()和ClientStub::setTransport()将传输从一个客户端存根移动到另一个客户端存根：
```cpp
RcfClient<I_AnotherInterface> client2( client.getClientStub().releaseTransport() );
client2.AnotherPrint("Hello World");
client.getClientStub().setTransport( client2.getClientStub().releaseTransport() );
client.Print("Hello World");
```
如果您希望使用相同的连接对不同的接口进行调用，这是非常有用的。

超时
有两个重要的超时设置会影响远程调用的执行方式。

第一个是连接超时，通过RCF::ClientStub::setConnectTimeoutMs()配置。此设置确定客户机在尝试建立到服务器的网络连接时将等待多长时间。您可以通过调用RCF::ClientStub::setConnectTimeoutMs()来为特定的RcfClient<>设置自定义连接超时，也可以通过调用RCF::globals()::setDefaultConnectTimeoutMs()来为所有RcfClient<>实例设置默认连接超时。

第二个是远程调用超时，通过RCF::ClientStub::setRemoteCallTimeoutMs()配置。此设置确定客户端连接后等待远程调用完成的时间。与连接超时一样，您可以通过调用RCF::ClientStub::setRemoteCallTimeoutMs()在特定的RcfClient<>上设置远程调用超时，也可以通过调用RCF::globals(). setdefaultremotecalltimeoutms()为所有RcfClient<>实例设置缺省远程调用超时。

两个超时都以毫秒为单位指定。

发出砰的
可以使用ClientStub::ping()方法来确定客户机是否具有到服务器的可行连接。ping的行为与没有in或out参数的远程调用完全一样，并且受到相同的超时限制。
```cpp
// Ping the server.
client.getClientStub().ping();
```
客户的进展通知
默认情况下，远程调用是同步执行的。这意味着在执行远程调用时，客户机线程被阻塞。不过，您可以将客户机配置为定期向自定义客户机进度回调函数报告进度。通过使用进度回调函数，您可以执行特定于应用程序的代码，例如显示进度条、刷新应用程序GUI或取消调用。
```cpp
std::uint32_t progressCallbackIntervalMs = 500;
auto progressCallback = [&](const RCF::RemoteCallProgressInfo& info, RCF::RemoteCallAction& action) 
{ 
    // Set to RCF::Rca_Cancel to cancel the call.
    action = RCF::Rca_Continue;
};
client.getClientStub().setRemoteCallProgressCallback(
    progressCallback,
    progressCallbackIntervalMs);
// While the call is executing, the progress callback will be called every 500ms.
client.Print("Hello World");
```
异步远程调用(参见异步调用)是在远程调用期间重新获得控制权的另一种方法。

远程调用模式
每个RCF客户机都有一个与之关联的远程调用模式。有两种基本的远程调用模式:

双向调用(RCF::Twoway)直到客户端收到来自服务器的响应才会完成。双向调用是默认的远程调用模式。
一旦请求被发送到服务器，单向调用(RCF::Oneway)就会完成。服务器不向单向调用发送任何响应。
可以使用RCF::ClientStub::setRemoteCallMode()设置远程调用模式。
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
在本例中，作为调用的一部分指定的远程调用模式覆盖RCF::ClientStub::setRemoteCallMode()设置的模式。

成批的单向调用
RCF支持批量单向调用，作为单向调用的扩展。

当RcfClient<>被配置为进行单向调用时，将为所进行的每个远程调用向服务器发送一条网络消息。如果正在进行大量调用，可以使用批处理单向调用将多个单向消息合并到单个网络消息中，同时仍然在服务器上按顺序执行远程调用。

根据发送的消息的数量和频率，这可能导致显著的性能提高。

要配置批处理单向调用，可以使用RCF::ClientStub::enableBatching()、RCF::ClientStub::disableBatching()和RCF::ClientStub::flushBatch()函数：
```cpp
client.getClientStub().enableBatching();
// Automatically send batch when message size approaches 50kb.
client.getClientStub().setMaxBatchMessageLength(1024*50);
for (std::size_t i=0; i<100; ++i)
{
    client.Print("Hello World");
}
// Send final batch.
client.getClientStub().flushBatch();
```
自定义请求和响应数据
通常，远程调用中从客户机传输到服务器的应用程序数据包含在远程调用的参数中。您还可以通过设置远程调用请求的用户数据字段来传输额外的信息。然后可以在服务器上访问这些数据以及远程调用的参数。

类似地，服务器代码可以设置远程调用响应的用户数据字段，并将该数据连同远程调用的参数一起传输回客户机。

用户数据必须指定为字符串。
```cpp
class PrintService
{
public:
    void Print(const std::string & s)
    {
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
用户数据字段可用于实现自定义身份验证方案，其中需要将令牌作为每个远程调用的一部分传递，但您不希望令牌成为远程调用接口的一部分。

复制语义
可以复制RcfClient<>对象。但是，RcfClient<>对象的每个副本都将建立到服务器的自己的网络连接。下面的代码将建立到服务器的3个网络连接：
```cpp
RcfClient<I_PrintService> client1(( RCF::TcpEndpoint(port) ));
RcfClient<I_PrintService> client2(client1);
RcfClient<I_PrintService> client3;
client3 = client1;
client1.Print("Hello World");
client2.Print("Hello World");
client3.Print("Hello World");
```