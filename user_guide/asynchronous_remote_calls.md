<!--
 * @Author: haoluo
 * @Date: 2019-07-16 10:04:11
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-16 10:09:00
 * @Description: file content
 -->
## 异步远程调用
异步调用
远程调用的异步调用允许您在一个线程上激发远程调用，并在另一个线程上异步完成远程调用。

异步调用是在RCF中使用RCF::Future<>类实现的。如果RCF::Future<>对象作为远程调用的返回值或参数之一提供，则远程调用将异步执行。

例如，给定这个RCF接口：
```cpp
RCF_BEGIN(I_PrintService, "I_PrintService")
    // Returns number of characters printed.
    RCF_METHOD_R1(int, Print, const std::string &)
RCF_END(I_PrintService)
```
，异步调用将由以下代码启动：
```cpp
    RcfClient<I_PrintService> client(( RCF::TcpEndpoint(port) ));
    RCF::Future<int> fRet = client.Print("Hello World");
```
执行此代码的线程将立即返回。守则随后可轮询完成：
```cpp
    // Poll until remote call completes.
    while (!fRet.ready()) {
        RCF::sleepMs(500);
    }
```
，或者可以等待完成：
```cpp
    // Wait for remote call to complete.
    fRet.wait();
```
一旦调用完成，返回值可以从相关的 `Future<>` 实例中恢复：
```cpp
    int charsPrinted = *fRet;
```
如果调用导致错误，则在取消对未来<>实例的引用时将抛出错误，如上所述。或者，可以通过调用RCF::Future<>::getAsyncException()来检索错误：
```cpp
    std::unique_ptr<RCF::Exception> ePtr = fRet.getAsyncException();
```
不是轮询或等待，发起异步远程调用的线程可以提供一个完成回调，当调用完成时，RCF运行时将在后台线程上调用该回调：
```cpp
typedef std::shared_ptr< RcfClient<I_PrintService> > PrintServicePtr;
void onCallCompleted(PrintServicePtr client, RCF::Future<int> fRet)
{
    std::unique_ptr<RCF::Exception> ePtr = fRet.getAsyncException();
    if (ePtr.get())
    {
        // Deal with any exception.
        // ...
    }
    else
    {
        int charsPrinted = *fRet;
        // ...
    }
}
```
```cpp
        RCF::Future<int> fRet;
        PrintServicePtr client( new RcfClient<I_PrintService>(RCF::TcpEndpoint(port)) );
        auto onCompletion = [=]() { onCallCompleted(client, fRet);  };
        fRet = client->Print( 
            RCF::AsyncTwoway(onCompletion), 
            "Hello World");
```
注意，未来<>参数作为参数传递给completion回调函数。未来<>对象是内部引用计数的对象，可以自由复制，同时仍然引用相同的底层值。

正在进行的异步调用，可以通过断开客户端来取消：
```cpp
    client.getClientStub().disconnect();
```
如果在进行异步调用时销毁了RcfClient，则连接将自动断开，任何异步操作都将被取消。

异步调度
在服务器端，RCF通常会在从传输中读取远程调用请求的同一服务器线程上发出远程调用。异步调度允许您将远程调用转移到其他线程，从而释放RCF服务器线程来处理其他远程调用。

RCF::RemoteCallContext<>类用于捕获远程调用的服务器端上下文。可以将RemoteCallContext<>对象复制到队列中，并存储在任意应用程序线程上，供以后执行。

RCF::RemoteCallContext<>对象是在对应的服务实现方法中创建的。为了进行比较，这里有一个非异步分派的Print()方法：
```cpp
RCF_BEGIN(I_PrintService, "I_PrintService")
    // Returns number of characters printed.
    RCF_METHOD_R1(int, Print, const std::string &)
RCF_END(I_PrintService)
class PrintService
{
public:
    int Print(const std::string & s)
    {
        std::cout << "I_PrintService service: " << s << std::endl;
        return (int) s.length();
    }
};
```
要替代异步调度调用，创建一个RCF::RemoteCallContext<> object in Print()，使用与方法签名对应的模板参数：
```cpp
class PrintService
{
public:
    typedef RCF::RemoteCallContext<int, const std::string&> PrintContext;
    int Print(const std::string & s)
    {
        // Capture current remote call context.
        PrintContext printContext(RCF::getCurrentRcfSession());
        // Start a new thread to dispatch the remote call.
        std::thread printAsyncThread([=]() { printAsync(printContext); });
        printAsyncThread.detach();
        return 0;
    }
    void printAsync(PrintContext printContext)
    {
        const std::string & s = printContext.parameters().a1.get();
        std::cout << "I_PrintService service: " << s << std::endl;
        printContext.parameters().r.set( (int) s.length() );
        printContext.commit();
    }
};
```
创建后，RCF::RemoteCallContext<>对象可以像任何其他c++对象一样存储和复制。当Print()函数返回时，RCF不会向客户机发送响应。只有在调用RCF::RemoteCallContext<>::commit()时才会发送响应。

parameters()提供对远程调用的所有参数的访问，包括返回值。上面的代码示例访问一个in-parameter：
```cpp
    const std::string & s = printContext.parameters().a1.get();
```
，然后设置一个out参数(在本例中为返回值)：
```cpp
    printContext.parameters().r.set( (int) s.length() );
```