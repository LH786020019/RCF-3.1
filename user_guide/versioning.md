<!--
 * @Author: haoluo
 * @Date: 2019-07-16 10:22:04
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-16 10:25:01
 * @Description: file content
 -->
## 版本控制
RCF提供了健壮的版本控制支持，允许您升级分布式组件，而不会破坏与以前部署的组件的运行时兼容性。

如果您编写的组件只与同一版本的对等组件通信，则无需担心版本控制。例如，如果所有组件都由相同的代码基构建并部署在一起，那么版本控制就不是问题。

但是，如果您的客户端需要与使用相同代码基的旧版本构建的服务器通信，并且服务器已经部署，那么您需要了解RCF如何处理版本控制。

与您可能遇到的其他RPC系统不同，RCF被设计成版本友好的。客户端与服务器之间的数据契约本质上定义如下:

RCF接口上的方法。
这些方法参数的顺序和类型。
参数的序列化函数。
RCF允许您在不使现有数据契约失效的情况下，扩展数据契约的所有这些方面。

接口版本控制
添加或删除方法
RCF接口中的每个方法都有一个与之关联的方法ID，客户端使用这个ID来标识特定的方法。接口上的第一个方法的方法ID为0，下一个方法的方法ID为1，依此类推。

在RCF接口的开始处或中间插入方法会更改现有方法ID，从而破坏与现有客户机和服务器的兼容性。为了保持兼容性，需要在RCF接口的末尾添加新方法：
```cpp
// Version 1
RCF_BEGIN(I_Calculator, "I_Calculator")
    RCF_METHOD_R2(double, add, double, double)
    RCF_METHOD_R2(double, subtract, double, double)
RCF_END(I_Calculator)
```
```cpp
// Version 2.
// * Clients compiled against this interface will be able to call add() and 
//   subtract() on servers compiled against the old interface.
//
// * Servers compiled against this interface will be able to take add() and 
//   subtract() calls from clients compiled against the old interface.
RCF_BEGIN(I_Calculator, "I_Calculator")
    RCF_METHOD_R2(double, add, double, double)
    RCF_METHOD_R2(double, subtract, double, double)
    RCF_METHOD_R2(double, multiply, double, double)
RCF_END(I_Calculator)
```
为了保存接口中其余方法的方法ID，只要在接口中保留一个占位符，就可以删除方法。
```cpp
// Version 1
RCF_BEGIN(I_Calculator, "I_Calculator")
    RCF_METHOD_R2(double, add, double, double)
    RCF_METHOD_R2(double, subtract, double, double)
RCF_END(I_Calculator)
```
```cpp
// Version 2. 
// * Clients compiled against this interface will be able to call subtract() 
//   on servers compiled against the old interface.
//
// * Servers compiled against this interface will be able to take subtract()
//   calls from clients compiled against the old interface (but not add() calls).
RCF_BEGIN(I_Calculator, "I_Calculator")
    RCF_METHOD_PLACEHOLDER()
    RCF_METHOD_R2(double, subtract, double, double)
RCF_END(I_Calculator)
```
添加或删除参数
参数可以添加到方法中，也可以从方法中删除，而不会破坏兼容性。RCF服务器和客户机忽略远程调用中传递的任何额外(冗余)参数，如果没有提供预期的参数，则默认初始化该参数。
```cpp
// Version 1
RCF_BEGIN(I_Calculator, "I_Calculator")
    RCF_METHOD_R2(double, add, double, double)
RCF_END(I_Calculator)
```
```cpp
// Version 2
// * Clients compiled against this interface will be able to call add() 
//   on servers compiled against the old interface (the server will ignore
//   the third parameter).
//
// * Servers compiled against this interface will be able to take add()
//   calls from clients compiled against the old interface (the third parameter
//   will be default initialized to zero).
RCF_BEGIN(I_Calculator, "I_Calculator")
    RCF_METHOD_R3(double, add, double, double, double)
RCF_END(I_Calculator)
```
同样，可以删除参数：
```cpp
// Version 1
RCF_BEGIN(I_Calculator, "I_Calculator")
    RCF_METHOD_R2(double, add, double, double)
RCF_END(I_Calculator)
```
```cpp
// Version 2
// * Clients compiled against this interface will be able to call add() 
//   on servers compiled against the old interface (the server will assume the
//   second parameter is zero).
//
// * Servers compiled against this interface will be able to take add()
//   calls from clients compiled against the old interface (the second parameter
//   from the client will be ignored).
RCF_BEGIN(I_Calculator, "I_Calculator")
    RCF_METHOD_R1(double, add, double)
RCF_END(I_Calculator)
```
注意，RCF封送in-parameters和out-parameters的顺序与它们在RCF_METHOD_XX()声明中出现的顺序相同。任何添加(或删除)的参数必须是最后编组的，否则将破坏兼容性。

重命名接口
RCF接口由它们的运行时名称标识，在RCF_BEGIN()宏的第二个参数中指定。只要保留这个名称，就可以更改接口的编译时名称，而不会破坏兼容性。
```cpp
// Version 1
RCF_BEGIN(I_Calculator, "I_Calculator")
    RCF_METHOD_R2(double, add, double, double)
RCF_END(I_Calculator)
```
```cpp
// Version 2
// * Clients compiled against this interface will be able to call add() 
//   on servers compiled against the old interface.
//
// * Servers compiled against this interface will be able to take add()
//   calls from clients compiled against the old interface.
RCF_BEGIN(I_CalculatorService, "I_Calculator")
    RCF_METHOD_R2(double, add, double, double)
RCF_END(I_CalculatorService)
```
存档的版本
在远程调用中传递的特定于应用程序的数据类型可能会随着时间而改变。随着这些数据类型的变化，它们的序列化函数也会发生变化。为了帮助应用程序维护序列化代码的向后兼容性，RCF提供了一个归档版本号概念。

存档版本号在每次远程调用时都从客户机传递到服务器，缺省情况下为零。要实现版本控制，您的应用程序需要管理存档版本，本质上是在对序列化代码进行中断更改时更新它。

要设置存档版本号，请在创建任何服务器或客户机之前调用RCF::setArchiveVersion()。

或者，您可以通过调用RCF::ClientStub::setArchiveVersion()和RCF::RcfServer::setArchiveVersion()来为单个客户机和服务器设置归档版本号。

当客户机连接到服务器时，会自动执行版本协商步骤，以确保连接使用两个组件都支持的最大存档版本号。

存档版本号用于序列化代码。通过调用SF:: archive::getArchiveVersion()，可以从序列化函数中检索正在使用的存档版本号：
```cpp
void serialize(SF::Archive & ar, MyClass & m)
{
    std::uint32_t archiveVersion = ar.getArchiveVersion();
    // ...
}
```
然后，序列化代码可以使用存档版本号的值来确定要序列化哪些成员。

例如，假设应用程序的第一个版本包含以下代码：
```cpp
class X
{
public:
    int mA = 0;
    void serialize(SF::Archive & ar)
    {
        ar & mA;
    }
};
RCF_BEGIN(I_Echo, "I_Echo")
    RCF_METHOD_R1(X, Echo, const X &)
RCF_END(I_Echo)
class EchoImpl
{
public:
    X Echo(const X & x) 
    { 
        return x; 
    }
};
```
```cpp
    //--------------------------------------------------------------------------
    // Accepting calls from other processes...
    EchoImpl echo;
    RCF::RcfServer server( RCF::TcpEndpoint(50001) );
    server.bind<I_Echo>(echo);
    server.start();
    //--------------------------------------------------------------------------
    // ... or making calls to other processes.
    RcfClient<I_Echo> client( RCF::TcpEndpoint(50002) );
    X x1;
    X x2;
    x2 = client.Echo(x1);
```
一旦这个版本发布，一个新的版本就准备好了，X类中添加了一个新成员：
```cpp
class X
{
public:
    int mA = 0;
    int mB = 0;
    void serialize(SF::Archive & ar)
    {
        // Retrieve archive version, to determine which members to serialize.
        std::uint32_t version = ar.getArchiveVersion();
        ar & mA;
        if (version >= 1)
        {
            ar & mB;
        }
    }
};
class EchoImpl
{
public:
    X Echo(const X & x) { return x; }
};
```