<!--
 * @Author: haoluo
 * @Date: 2019-07-17 19:08:43
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-17 19:17:39
 * @Description: file content
 -->
## Server 端会话

### 1. Server 端会话对象 —— Server
这个示例演示了如何使用 `RCF::RcfSession::getSessionObject()` 和 `RCF::RcfServer::getServerObject()` 来维护连接会话和 server 会话。
```cpp
#include <iostream>
#include <RCF/RCF.hpp>
// 定义 I_PrintService RCF 接口。
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V2(void, Print, const std::string&, const std::string &)
RCF_END(I_PrintService)
// 该类表示与到该 server 的单个网络连接相关联的状态。
class ConnectionSession{
public:
    int mCallsMade = 0;
};
// 该类表示与该 server 上的一个 server 会话相关联的状态。
// 它没有连接到任何特定的网络连接。
class ServerSession{
public:
    std::string mUserName;
    int mCallsMade = 0;
};
typedef std::shared_ptr<ServerSession> ServerSessionPtr;

class PrintService{
public:
    void Print(const std::string& userName, const std::string & msg){
        RCF::RcfSession & rcfSession = RCF::getCurrentRcfSession();
        // 更新连接会话
        ConnectionSession & session = rcfSession.getSessionObject<ConnectionSession>(true);
        ++session.mCallsMade;
        // 更新 server 会话
        RCF::RcfServer & server = rcfSession.getRcfServer();
        ServerSessionPtr sessionPtr = server.getServerObject<ServerSession>(userName, 60 * 1000);
        sessionPtr->mUserName = userName;
        ++sessionPtr->mCallsMade;
        std::cout << std::endl;
        std::cout << "I_PrintService service (" << userName << "): " << msg << std::endl;
        std::cout << "Print() calls made on this connection: " << session.mCallsMade << std::endl;
        std::cout << "Print() calls made by this user: " << sessionPtr->mCallsMade << std::endl;
    }
};
int main(){
    try{
        RCF::RcfInit rcfInit;
        PrintService printService;
        RCF::RcfServer server(RCF::TcpEndpoint("127.0.0.1", 50001));
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

### 2. Server 端会话对象 —— Client
这个示例演示了 client 调用 server 并在 server 上维护连接会话和 server 会话。
```cpp
#include <iostream>
#include <RCF/RCF.hpp>
// 定义 RCF 接口
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V2(void, Print, const std::string&, const std::string &)
RCF_END(I_PrintService)
int main(){
    try{
        RCF::RcfInit rcfInit;
        std::string userName = "Joe Bloggs";
        for ( int i = 0; i < 3; ++i ) {
            RcfClient<I_PrintService> client(RCF::TcpEndpoint("127.0.0.1", 50001));
            for ( int i = 0; i < 5; ++i ) {
                client.Print(userName, "Hello World");
            }
        }
    } catch ( const RCF::Exception & e ) {
        std::cout << "Error: " << e.getErrorMessage() << std::endl;
    }
    return 0;
}
```