<!--
 * @Author: haoluo
 * @Date: 2019-07-16 10:04:11
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-17 12:41:16
 * @Description: file content
 -->
## 异步远程调用(Asynchronous Remote Calls)
### 1. 异步调用(Asynchronous Invocation)
远程调用的异步调用允许您在一个线程上激发远程调用，并在另一个线程上异步完成远程调用。

异步调用是在 RCF 中使用 [RCF::Future<>](http://www.deltavsoft.com/doc/class_r_c_f_1_1_future.html) 类实现的。如果 [RCF::Future<>](http://www.deltavsoft.com/doc/class_r_c_f_1_1_future.html) 对象作为远程调用的返回值或参数之一提供，则远程调用将异步执行。

例如，给定这个 RCF 接口：
```cpp
RCF_BEGIN(I_PrintService, "I_PrintService")
    // 返回打印的字符数
    RCF_METHOD_R1(int, Print, const std::string &)
RCF_END(I_PrintService)
```
，异步调用将由以下代码启动：
```cpp
    RcfClient<I_PrintService> client(( RCF::TcpEndpoint(port) ));
    RCF::Future<int> fRet = client.Print("Hello World");
```
执行此代码的线程将立即返回。此代码随后可轮询来完成：
```cpp
    // 在远程调用完成之前进行轮询
    while (!fRet.ready()) {
        RCF::sleepMs(500);
    }
```
，或者可以等待来完成：
```cpp
    // 等待远程调用完成
    fRet.wait();
```
一旦调用完成，返回值可以从相关的 [RCF::Future<>](http://www.deltavsoft.com/doc/class_r_c_f_1_1_future.html) 实例中恢复：
```cpp
    int charsPrinted = *fRet;
```
如果调用导致错误，则在取消对 [RCF::Future<>](http://www.deltavsoft.com/doc/class_r_c_f_1_1_future.html) 实例的引用时将抛出错误，如上所述。或者，可以通过调用 `RCF::Future<>::getAsyncException()` 来检索错误：
```cpp
    std::unique_ptr<RCF::Exception> ePtr = fRet.getAsyncException();
```
代替使用轮询或等待，激发一个异步远程调用的线程可以提供一个完成回调(`completion callback`)，当调用完成时，RCF 运行时将在一个后台线程上调用该回调：
```cpp
typedef std::shared_ptr< RcfClient<I_PrintService> > PrintServicePtr;
void onCallCompleted(PrintServicePtr client, RCF::Future<int> fRet){
    std::unique_ptr<RCF::Exception> ePtr = fRet.getAsyncException();
    if (ePtr.get()) {
        // Deal with any exception.
        // ...
    } else {
        int charsPrinted = *fRet;
        // ...
    }
}
```
```cpp
        RCF::Future<int> fRet;
        PrintServicePtr client( new RcfClient<I_PrintService>(RCF::TcpEndpoint(port)) );
        auto onCompletion = [=]() { onCallCompleted(client, fRet);  };
        fRet = client->Print(RCF::AsyncTwoway(onCompletion), "Hello World");
```
注意，`Future<>` 参数作为参数传递给完成回调函数。`Future<>` 对象是内部引用计数的对象，可以自由复制，同时仍然引用相同的底层值。

正在进行的一个异步调用，可以通过断开 client 来取消：
```cpp
    client.getClientStub().disconnect();
```
如果在进行一个异步调用时销毁了 `RcfClient`，则连接将自动断开，任何异步操作都将被取消。

### 2. 异步调度(Asynchronous Dispatching)
在 server 端，RCF 通常会在从传输中读取远程调用请求的同一 server 线程上调度一个远程调用。异步调度允许您将远程调用转移到其他线程，从而释放 RCF server 线程来处理其他远程调用。

[RCF::RemoteCallContext<>](http://www.deltavsoft.com/doc/class_r_c_f_1_1_remote_call_context.html) 类用于捕获一个远程调用的 server 端上下文。可以将 [RCF::RemoteCallContext<>](http://www.deltavsoft.com/doc/class_r_c_f_1_1_remote_call_context.html) 对象复制到队列中，并存储在任意应用程序线程上，供以后执行。

[RCF::RemoteCallContext<>](http://www.deltavsoft.com/doc/class_r_c_f_1_1_remote_call_context.html) 对象是在对应的服务实现方法中创建的。为了进行比较，这里有一个非异步调度的 `Print()` 方法：
```cpp
RCF_BEGIN(I_PrintService, "I_PrintService")
    // 返回打印的字符数
    RCF_METHOD_R1(int, Print, const std::string &)
RCF_END(I_PrintService)
class PrintService{
public:
    int Print(const std::string & s){
        std::cout << "I_PrintService service: " << s << std::endl;
        return (int) s.length();
    }
};
```
要替代异步地调度调用，在 `Print()` 中创建一个 [RCF::RemoteCallContext<>](http://www.deltavsoft.com/doc/class_r_c_f_1_1_remote_call_context.html) 对象，使用与方法签名对应的模板参数：
```cpp
class PrintService{
public:
    typedef RCF::RemoteCallContext<int, const std::string&> PrintContext;
    int Print(const std::string & s){
        // 捕获当前远程调用上下文。
        PrintContext printContext(RCF::getCurrentRcfSession());
        // 启动一个新线程来调度远程调用。
        std::thread printAsyncThread([=]() { printAsync(printContext); });
        printAsyncThread.detach();
        return 0;
    }
    void printAsync(PrintContext printContext){
        const std::string & s = printContext.parameters().a1.get();
        std::cout << "I_PrintService service: " << s << std::endl;
        printContext.parameters().r.set( (int) s.length() );
        printContext.commit();
    }
};
```
创建后，[RCF::RemoteCallContext<>](http://www.deltavsoft.com/doc/class_r_c_f_1_1_remote_call_context.html) 对象可以像任何其他 C++ 对象一样存储和复制。当 `Print()` 函数返回时，RCF 不会向 client 发送响应。只有在调用 `RCF::RemoteCallContext<>::commit()` 时才会发送响应。

[RCF::RemoteCallContext::parameters()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_remote_call_context.html#a21d3e4331f8093a0ce66643311819402) 提供对远程调用的所有参数的访问，包括返回值。上面的代码示例访问一个输入参数：
```cpp
    const std::string & s = printContext.parameters().a1.get();
```
，然后设置一个输出参数(在本例中为返回值)：
```cpp
    printContext.parameters().r.set( (int) s.length() );
```