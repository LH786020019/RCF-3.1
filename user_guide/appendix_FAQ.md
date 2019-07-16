<!--
 * @Author: haoluo
 * @Date: 2019-07-16 10:38:20
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-16 10:42:15
 * @Description: file content
 -->
## 附录 —— 常见问题解答
建筑
为什么我的程序试图链接到Boost库?
如果您已经定义了RCF_FEATURE_BOOST_SERIALIZATION，那么RCF将需要链接到Boost。序列化的图书馆。

RCF不链接到任何其他Boost库。

Boost库具有自动链接功能，可能会导致链接器查找要链接的错误文件。您可以定义BOOST_ALL_NO_LIB，然后显式地告诉链接器要链接到哪些文件。

我可以把RCF编译成DLL吗?
是的。要从DLL (Windows)或共享库(Unix)导出RCF函数，需要在构建DLL或共享库时定义RCF_BUILD_DLL。

为什么我有时会得到编译器错误当我包括<windows。< span="">RCF头文件前的h> ?</windows。<>
历史上，Windows platform headers < Windows存在一些标题排序问题。h >和< winsock2.h >。包括<窗口。在默认情况下，h>将包含一个较老版本的Winsock，因此不可能随后包含<winsock2。< span="">h>在同一个平移单位。</winsock2。<>

这个问题最简单的解决方法是在包含<windows.h>之前定义WIN32_LEAN_AND_MEAN。</windows.h>

RCF是否在Visual c++的第4级免费编译警告?
是的，如果以下警告被禁用：
```output
C4510 'class' : default constructor could not be generated
C4511 'class' : copy constructor could not be generated
C4512 'class' : assignment operator could not be generated
C4127 conditional expression is constant
```
我可以在XYZ平台上运行RCF吗?
可能，但是您可能需要自己对RCF做一些小的修改，以适应平台特定的问题，比如要包含哪些平台头文件。

为什么会出现链接器错误?
在Windows上，如果使用TCP传输，则需要链接到ws2_32.lib。在Unix平台上，需要链接到libnsl或libsocket等库。

RCF支持64位编译器吗?
是的。

RCF是否支持基于Windows的Unicode构建?
是的。

为什么外部序列化函数会出现重复的符号链接器错误?
如果在头文件中定义外部序列化函数，而不使用内联修饰符，并将它们包含在两个或多个源文件中，则会得到关于重复符号的链接器错误。解决方案是添加一个内联修饰符：
```cpp
// X.hpp
inline void serialize(SF::Archive & ar, X & x)
{
    ...
}
```
，或在头文件中声明序列化函数并在源文件中定义它：
```cpp
// X.hpp
void serialize(SF::Archive & ar, X & x);
```
```cpp
// X.cpp
void serialize(SF::Archive & ar, X & x)
{
    ...
}
```
如何减少应用程序的构建时间?
如果您将RCF头包含到您自己常用的应用程序头中，您可能会注意到构建时间的增加，因为编译器将为包含RCF头的每个源文件解析一次RCF头。

您应该只在需要时包含RCF头—换句话说，只将它们包含到使用RCF功能的源文件中。在应用程序头文件中，应该能够使用正向声明，而不是包含相应的头文件。

例如，如果您定义了一个类X与RcfClient<>成员，您可以转发声明RcfClient<>，然后使用一个指针为成员：
```cpp
// X.h
template<typename T>
class RcfClient;
class SomeInterface;
typedef RcfClient<SomeInterface> MyRcfClient;
typedef std::shared_ptr<MyRcfClient> MyRcfClientPtr;
class RcfServer;
typedef std::shared_ptr<RcfServer> RcfServerPtr;
// Application specific class that holds a RcfClient and a RcfServer.
class X
{  
    X();
    MyRcfClientPtr mClientPtr;
    RcfServerPtr mServerPtr;
};
// X.cpp
#include "X.h"
#include <RCF/RcfClient.hpp>
#include <RCF/RcfServer.hpp>
X::X() : 
    mClientPtr( new RcfClient<SomeInterface>(...) ), 
    mRcfServerPtr( new RcfServer(...) ) 
{}
```
```cpp
// X.h
template<typename T>
class RcfClient;
class SomeInterface;
typedef RcfClient<SomeInterface> MyRcfClient;
typedef std::shared_ptr<MyRcfClient> MyRcfClientPtr;
class RcfServer;
typedef std::shared_ptr<RcfServer> RcfServerPtr;
// Application specific class that holds a RcfClient and a RcfServer.
class X
{  
    X();
    MyRcfClientPtr mClientPtr;
    RcfServerPtr mServerPtr;
};
// X.cpp
#include "X.h"
#include <RCF/RcfClient.hpp>
#include <RCF/RcfServer.hpp>
X::X() : 
    mClientPtr( new RcfClient<SomeInterface>(...) ), 
    mRcfServerPtr( new RcfServer(...) ) 
{}
```
然后，您可以在应用程序的任何地方包含X.h，而不需要包含任何RCF头。

平台
为什么我用完了Windows XP上的套接字句柄?
无论何时建立一个传出TCP连接，都必须为连接分配一个本地端口号。在Windows XP中，这些本地端口(有时称为临时端口)默认是从大约4000个端口号的范围分配的。

如果您使用TCP端点创建许多RcfClient<>对象，您最终将耗尽可用端口，因为在连接关闭后，Windows会占用它们一小段时间。

您应该尽可能少地使用TCP连接(重用RcfClient<>对象，而不是创建新的对象)。Windows XP上还有一些注册表设置可以缓解这个问题。找到以下键:

HKEY_LOCAL_MACHINE \ SYSTEM \ CurrentControlSet \ Tcpip \ \服务参数

,并设置

TcpNumConnections = 0x800, MaxUserPort = 65534

重新启动后，系统将允许扩展临时端口的范围。

这个问题只与旧版本的Windows有关。Windows Vista和后来使用的临时端口范围扩大。

为什么我的检漏器认为RCF有泄漏?
可能是因为还没有执行RCF::deinit()。

RCF的TCP服务器实现是否基于I/O完成端口?
在Windows上，RCF使用I/O完成端口。看到可伸缩性。

RCF是否支持共享内存传输?
对于Windows上的本地RPC, RCF支持由共享内存支持的命名管道传输(RCF::Win32NamedPipeEndpoint)。

对于Unix上的本地RPC, RCF支持Unix本地套接字传输(RCF::UnixLocalEndpoint)。

RCF支持IPv6吗?
是的，请参阅IPv4/IPv6。

编程
当远程调用正在进行时，如何防止用户界面失去响应?
要么在非UI线程上运行远程调用，要么使用进度回调以短时间间隔重新绘制UI。参见客户端进度通知。

如何取消长时间运行的客户端调用?
使用进度回调(参见客户端进度通知)。您可以将回调配置为在任何给定频率调用，当您想取消调用时，抛出异常。

如何在远程调用中停止服务器?
您不能在远程调用中调用RCF::RcfServer::stop()，因为stop()调用将等待所有工作线程退出，包括调用stop()的线程，从而导致死锁。

如果你真的需要在远程调用中停止服务器，你可以启动一个新的线程：
```cpp
RCF_BEGIN(I_Echo, "I_Echo")
    RCF_METHOD_R1(std::string, Echo, const std::string &)
RCF_END(I_Echo)
class EchoImpl
{
public:
    std::string Echo(const std::string &s)
    {
        if (s == "stop")
        {
            // Spawn a temporary thread to stop the server.
            RCF::RcfServer * pServer = & RCF::getCurrentRcfSession().getRcfServer();
            RCF::Thread t( [=]() { pServer->stop(); } );
            t.detach();
        }
        return s;
    }
};
```
如何使远程访问的函数成为私有的?
让RcfClient<>成为你的实现类的朋友：
```cpp
    RCF_BEGIN(I_Echo, "I_Echo")
        RCF_METHOD_R1(std::string, Echo, const std::string &)
    RCF_END(I_Echo)
    class EchoImpl
    {
    private:
        friend RcfClient<I_Echo>;
        std::string Echo(const std::string &s)
        {
            return s;
        }
    };
```
我如何使用一个TCP连接与几个RcfClient<>实例?
您可以将网络连接从一个RcfClient<>移动到另一个RcfClient。看到交通访问。

如何强制断开客户机与服务器的连接?
在服务器实现中调用RCF::getCurrentRcfSession().disconnect()。

RcfClient<>对象何时连接到服务器?
RcfClient<>只会在您启动远程调用后建立网络连接。如果您想建立一个网络连接而不需要实际进行远程调用，那么使用RCF::ClientStub::connect()。

如何确定客户机从哪个IP地址连接?
在服务器实现中调用RCF::getCurrentRcfSession(). getclientaddress()。

如何从服务器端代码检测客户机断开连接?
当客户机断开连接时，服务器上关联的RCF::RcfSession将被销毁。可以使用RcfSession::setOnDestroyCallback()，在发生这种情况时通知应用程序代码。
```cpp
void onClientDisconnect(RCF::RcfSession & session)
{
    // ...
}
class ServerImpl
{
    void SomeFunc()
    {
        // From within a remote call, register a callback to be called when this RcfSession is destroyed.
        auto onDestroy = [&](RCF::RcfSession& session) { onClientDisconnect(session); };
        RCF::getCurrentRcfSession().setOnDestroyCallback(onDestroy);
    }
};
```
我可以通过防火墙使用发布/订阅吗?
是的。只要订阅者能够发起到发布者的连接，它就会接收已发布的消息。发布服务器将永远不会启动任何返回到订阅服务器的网络连接。

如何知道订阅者是否收到已发布的消息?
发布者不能知道这一点，因为消息是使用单向语义发布的。

订阅者如何知道发布者是否已停止发布?
使用订阅服务器断开连接通知。看到用户。

还可以使用RCF::Subscription::isConnected()轮询连接性。

为什么我不能在RCF接口中使用指针?
指针不能用作RCF接口中的返回类型，因为没有安全的方法来封送它们。

指针可以用作RCF接口中的参数，尽管大多数情况下，您最好使用引用，或者c++中可用的一种智能指针类型。

如何在机器上的第一个可用端口上启动TCP服务器?
在传递给RCF::RcfServer构造函数的RCF::TcpEndpoint对象中指定端口号为零。启动服务器后，通过调用RcfServer::getIpServerTransport(). getport()检索端口号。

如何使用相同的RCF接口公开多个服务对象?
使用服务绑定名称。参见添加服务绑定。

为什么我的服务器只能从本地机器访问，而不能通过网络访问?
在传递给RCF::RcfServer的RCF::TcpEndpoint中，您需要指定0.0.0.0作为IP地址，以允许客户机通过任何网络接口访问它。默认情况下使用的是127.0.0.1，它只允许本地机器上的客户机连接。

为什么我的程序在退出时崩溃或断言?
可能是因为您的程序有一个全局静态对象，其析构函数试图在RCF被取消初始化之后销毁RcfClient<>或RcfServer对象(或其他一些RCF对象)。

确保在反初始化RCF之前销毁所有RCF对象。

如何序列化枚举?
RCF将自动序列化和反序列化c++ 03枚举，作为整数表示。

c++ 11枚举类需要一个助手宏SF_SERIALIZE_ENUM_CLASS()(参见标准c++类型)。

我可以使用SF来序列化对象和文件吗?
是的，请参阅从磁盘到磁盘的序列化。

如何使用RCF发送文件?
看到文件传输。

如何访问RCF服务器使用的内部asio::io_service ?
您可以调用AsioServerTransport::getIoService()：
```cpp
#include <RCF/AsioServerTransport.hpp>
RCF::RcfServer server( RCF::TcpEndpoint(0) );
RCF::ServerTransport & transport = server.getServerTransport();
RCF::AsioServerTransport & asioTransport = dynamic_cast<RCF::AsioServerTransport &>(transport);
boost::asio::io_service & ioService = asioTransport.getIoService();
```
当RcfServer停止时，io_service将被销毁。

如何在不更改RCF接口的情况下将安全令牌从客户机传递到服务器?
您可以使用自定义请求用户数据(RCF::ClientStub::setRequestUserData()， RCF::RcfSession::getRequestUserData())，将特定于应用程序的数据从客户机传递到服务器。参见自定义请求和响应数据。

我可以在Linux和Windows之间发送std::wstring对象吗?
是的。wstring对象假设在Linux上用UTF-32表示，在Windows上用UTF-16表示，RCF在序列化时将它们编码为UTF-8。

我可以在Linux和Windows之间发送UTF-8编码的std::string对象吗?
是的。RCF将std::string序列化为一个由8位字符组成的序列，所以编码是ASCII、ISO-8859-1、UTF-8还是其他任何东西都无关紧要。

杂项
为什么文档中的许多示例中都有双括号?
以下代码片段将导致编译器错误：
```cpp
        int port = 0;
        RcfClient<I_Echo> client( RCF::TcpEndpoint(port) );
        RCF::RcfServer server( RCF::TcpEndpoint(port) );
        // Will get compiler errors on both of these lines...
        client.getClientStub();
        server.start();
```
由于c++语言的特殊性，客户机和服务器的声明实际上被解释为函数声明，使用一个名为port的RCF::TcpEndpoint参数。c++编译器以这种方式解释声明，以保持与C的向后兼容性。

为了消除这种歧义，需要在RCF::TcpEndpoint周围加上括号：
```cpp
        int port = 0;
        RcfClient<I_Echo> client(( RCF::TcpEndpoint(port) ));
        RCF::RcfServer server(( RCF::TcpEndpoint(port) ));
        // Now its OK.
        client.getClientStub();
        server.start();
```
c++语言的这种怪癖有时被称为“c++最烦人的解析”。