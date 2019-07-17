<!--
 * @Author: haoluo
 * @Date: 2019-07-17 19:40:19
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-17 19:44:10
 * @Description: file content
 -->
## 异步远程调用
### 1. 远程调用的异步调度(Asynchronous Dispatching of Remote Calls)
这个示例演示了远程调用的异步 server 端调度。
```cpp
#include <iostream>
#include <deque>
#include <RCF/RCF.hpp>
// Define RCF interface.
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_R1(int, Print, const std::string &)
RCF_END(I_PrintService)
// This class maintains a dedicated thread for processing of Print() calls.
class PrintService{
public:
    typedef RCF::RemoteCallContext<int, const std::string &> PrintCall;
    int Print(const std::string & msg){
        // Capture the remote call context and queue it in mPrintCalls.
        RCF::Lock lock(mPrintCallsMutex);
        mPrintCalls.push_back(PrintCall(RCF::getCurrentRcfSession()));
        // As the remote call has been captured, the return value here is no longer relevant.
        return 0;
    }
    PrintService(){
        // Start the asynchronous printing thread.
        mStopFlag = false;
        mPrinterThreadPtr.reset(new RCF::Thread(
            [&]() { processPrintCalls();  }));
    }
    ~PrintService(){
        // Stop the asynchronous printing thread.
        mStopFlag = true;
        mPrinterThreadPtr->join();
    }
private:
    // Queue of remote calls.
    RCF::Mutex              mPrintCallsMutex;
    std::deque<PrintCall>   mPrintCalls;
    // Asynchronous printing thread.
    RCF::ThreadPtr          mPrinterThreadPtr;
    volatile bool           mStopFlag;
    void processPrintCalls(){
        // Once a second, process all queued Print() calls.
        while ( !mStopFlag ) {
            Sleep(100);
            // Retrieve all queued print calls.
            std::deque<PrintCall> printCalls;
            {
                RCF::Lock lock(mPrintCallsMutex);
                printCalls.swap(mPrintCalls);
            }
            // Process them.
            for ( std::size_t i = 0; i < printCalls.size(); ++i ) {
                PrintCall & printCall = printCalls[i];
                // Retrieve a parameter.
                const std::string & stringToPrint = printCall.parameters().a1.get();
                std::cout << "I_PrintService service: " << stringToPrint << std::endl;
                // Set a return value.
                printCall.parameters().r.set( (int) stringToPrint.size());
                // Send the remote call response back to the client.
                printCall.commit();
            }
        }
    }
};
int main(){
    try{
        RCF::RcfInit rcfInit;
        RCF::RcfServer server(RCF::TcpEndpoint("127.0.0.1", 50001));
        PrintService printService;
        server.bind<I_PrintService>(printService);
        server.start();
        std::cout << "Press Enter to exit..." << std::endl;
        std::cin.get();
    } catch ( const RCF::Exception & e ) {
        std::cout << "Error: " << e.getErrorMessage() << std::endl;
    }
    return 0;
}
```

### 2. 远程调用的异步调用(Asynchronous Invocation of Remote Calls)
此示例演示远程调用的异步 client 调用。
```cpp
#include <iostream>
#include <RCF/RCF.hpp>
// Define RCF interface.
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_R1(int, Print, const std::string &)
RCF_END(I_PrintService)
typedef std::shared_ptr< RcfClient<I_PrintService> > PrintServicePtr;
// Remote call completion handler.
void onPrintCompleted(PrintServicePtr client, RCF::Future<int> fRet){
    std::unique_ptr<RCF::Exception> ePtr = fRet.getAsyncException();
    if ( ePtr.get() ) {
        std::cout << "Print() returned an exception: " << ePtr->getErrorMessage() << std::endl;
    } else {
        std::cout << "Print() returned: " << *fRet << std::endl;
    }
}
int main(){
    try{
        RCF::RcfInit rcfInit;
        RcfClient<I_PrintService> client(RCF::TcpEndpoint("127.0.0.1", 50001));
        // Synchronous call.
        std::cout << std::endl;
        std::cout << "Synchronous call:" << std::endl;
        int ret = client.Print("Hello World");
        std::cout << "Print() returned: " << ret << std::endl;
        {
            // Asynchronous call with polling.
            std::cout << std::endl;
            std::cout << "Asynchronous call with polling:" << std::endl;
            RCF::Future<int> fRet = client.Print("Hello World");
            while ( !fRet.ready() ) {
                RCF::sleepMs(500);
            }
            std::unique_ptr<RCF::Exception> ePtr = fRet.getAsyncException();
            if ( ePtr ) {
                std::cout << "Print() returned an exception: " << ePtr->getErrorMessage() << std::endl;
            } else {
                std::cout << "Print() returned: " << *fRet << std::endl;
            }
        }
        {
            PrintServicePtr clientPtr(new RcfClient<I_PrintService>(RCF::TcpEndpoint(50001)));
            // Asynchronous call with completion callback.
            std::cout << std::endl;
            std::cout << "Asynchronous call with completion callback:" << std::endl;
            RCF::Future<int> fRet;
            auto onCompletion = [=]() { onPrintCompleted(clientPtr, fRet); };
            fRet = clientPtr->Print(
                RCF::AsyncTwoway(onCompletion),
                "Hello World");
            // clientPtr goes out of scope here, but a reference to it is still held in onCompletion, and will
            // be passed to the completion handler when the call completes.
        }
        RCF::sleepMs(1000);
        std::cout << std::endl;
        std::cout << "Press Enter to exit..." << std::endl;
        std::cin.get();
    } catch ( const RCF::Exception & e ) {
        std::cout << "Error: " << e.getErrorMessage() << std::endl;
    }
    return 0;
}
```