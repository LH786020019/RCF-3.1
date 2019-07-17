<!--
 * @Author: haoluo
 * @Date: 2019-07-17 18:59:56
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-17 19:07:58
 * @Description: file content
 -->
## TCP Server 和 Client
### 1. TCP Server
这个示例演示了一个简单的 TCP server。
```cpp

#include <iostream>
#include <RCF/RCF.hpp>
// 定义 I_PrintService RCF 接口。
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
// I_PrintService RCF 接口的 server 实现。
class PrintService{
public:
    void Print(const std::string & s){
        std::cout << "I_PrintService service: " << s << std::endl;
    }
};
int main(){
    try{
        // 初始化 RCF.
        RCF::RcfInit rcfInit;
        // 实例化一个 RCF server
        RCF::RcfServer server(RCF::TcpEndpoint("127.0.0.1", 50001));
        // 绑定 I_PrintService 接口
        PrintService printService;
        server.bind<I_PrintService>(printService);
        // 启动 server
        server.start();
        std::cout << "Press Enter to exit..." << std::endl;
        std::cin.get();
    } catch ( const RCF::Exception & e ) {
        std::cout << "Error: " << e.getErrorMessage() << std::endl;
    }
    return 0;
}
```
### 2. TCP Client
这个示例演示了一个 TCP client。它打算在上面的 TCP server 上运行。
```cpp
#include <iostream>
#include <RCF/RCF.hpp>
// 定义 I_PrintService RCF 接口。
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
int main(){
    try{
        // 初始化 RCF
        RCF::RcfInit rcfInit;
        std::cout << "Calling the I_PrintService Print() method." << std::endl;
        
        // 实例化一个 RCF client
        RcfClient<I_PrintService> client(RCF::TcpEndpoint("127.0.0.1", 50001));
        // 连接到 server 并调用 Print() 方法。
        client.Print("Hello World");
    } catch ( const RCF::Exception & e ) {
        std::cout << "Error: " << e.getErrorMessage() << std::endl;
    }
    return 0;
}
```
### 3. 合并 TCP Server 和 Client
这个示例演示了一个简单的 TCP client 和 server，在单个进程中进行通信。
```cpp

#include <iostream>
#include <RCF/RCF.hpp>
// 定义 I_PrintService RCF 接口。
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
// I_PrintService RCF 接口的 server 实现。
class PrintService{
public:
    void Print(const std::string & s){
        std::cout << "I_PrintService service: " << s << std::endl;
    }
};
int main(){
    try{
        // 初始化 RCF.
        RCF::RcfInit rcfInit;
        // 实例化一个 RCF server.
        RCF::RcfServer server( RCF::TcpEndpoint("127.0.0.1", 50001) );
        // 绑定 I_PrintService 接口
        PrintService printService;
        server.bind<I_PrintService>(printService);
        
        // 启动 server.
        server.start();
        std::cout << "Calling the I_PrintService Print() method." << std::endl;
        // 实例化一个 RCF client.
        RcfClient<I_PrintService> client( RCF::TcpEndpoint("127.0.0.1", 50001) );
        // 连接到 server 并调用 Print() 方法。
        client.Print("Hello World");
    } catch ( const RCF::Exception & e ) {
        std::cout << "Error: " << e.getErrorMessage() << std::endl;
    }
    return 0;
}
```