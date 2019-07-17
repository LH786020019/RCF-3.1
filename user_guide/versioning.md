<!--
 * @Author: haoluo
 * @Date: 2019-07-16 10:22:04
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-17 15:42:38
 * @Description: file content
 -->
## 版本控制
RCF 提供了健壮的版本控制支持，允许您升级分布式组件，而不会破坏与以前部署的组件的运行时兼容性。

如果您编写的组件只与同一版本的对等组件通信，则无需担心版本控制。例如，如果所有组件都由相同的代码基构建并部署在一起，那么版本控制就不是问题。

但是，如果您的 client 需要与使用相同代码基的旧版本构建的 server 通信，并且 server 已经部署，那么您需要了解 RCF 如何处理版本控制。

与您可能遇到的其他 RPC 系统不同，RCF 被设计成版本友好的。Client 与 server 之间的数据契约本质上定义如下:
- RCF 接口上的方法。
- 这些方法参数的顺序和类型。
- 参数的序列化函数。

RCF 允许您在不使现有数据契约失效的情况下，扩展数据契约的所有这些方面。

### 1. 接口版本控制
#### 1.1 添加或删除方法
一个 RCF 接口中的每个方法都有一个与之关联的方法 ID，client 使用这个 ID 来标识特定的方法。接口上的第一个方法的方法 ID 为 0，下一个方法的方法 ID 为 1，依此类推。

在 RCF 接口的开始处或中间插入方法会更改现有方法 ID，从而破坏与现有 client 和 server 的兼容性。为了保持兼容性，需要在 RCF 接口的末尾添加新方法：
```cpp
// Version 1
RCF_BEGIN(I_Calculator, "I_Calculator")
    RCF_METHOD_R2(double, add, double, double)
    RCF_METHOD_R2(double, subtract, double, double)
RCF_END(I_Calculator)
```
```cpp
// Version 2.
// * 根据此接口编译的 client 将能够在根据旧接口编译的 server 上调用 add() 和 subtract()。
//
// * 根据此接口编译的 server 将能够从根据旧接口编译的 client 获得 add() 和 subtract() 调用。
RCF_BEGIN(I_Calculator, "I_Calculator")
    RCF_METHOD_R2(double, add, double, double)
    RCF_METHOD_R2(double, subtract, double, double)
    RCF_METHOD_R2(double, multiply, double, double)
RCF_END(I_Calculator)
```
为了保存接口中其余方法的方法 ID，只要在接口中保留一个占位符，就可以删除方法。
```cpp
// Version 1
RCF_BEGIN(I_Calculator, "I_Calculator")
    RCF_METHOD_R2(double, add, double, double)
    RCF_METHOD_R2(double, subtract, double, double)
RCF_END(I_Calculator)
```
```cpp
// Version 2. 
// * 根据这个接口编译的 client 将能够调用针对旧接口编译的 server 上的 subtract()。
//
// * 根据此接口编译的 server 将能够从根据旧接口编译的 client 中获得 subtract() 调用
// （ 但不能获得 add() 调用 ）。
RCF_BEGIN(I_Calculator, "I_Calculator")
    RCF_METHOD_PLACEHOLDER()
    RCF_METHOD_R2(double, subtract, double, double)
RCF_END(I_Calculator)
```
#### 1.2 添加或删除参数
参数可以添加到方法中，也可以从方法中删除，而不会破坏兼容性。RCF server 和 client 忽略远程调用中传递的任何额外(冗余)参数，如果没有提供预期的参数，则默认初始化该参数。
```cpp
// Version 1
RCF_BEGIN(I_Calculator, "I_Calculator")
    RCF_METHOD_R2(double, add, double, double)
RCF_END(I_Calculator)
```
```cpp
// Version 2
// * 根据此接口编译的 client 将能够调用针对旧接口编译的 server 上的 add()(server 将忽略第三个参数)。
//
// * 根据此接口编译的 server 将能够接受根据旧接口编译的 client 的 add() 调用(第三个参数将默认初始化为零)。
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
// * 根据这个接口编译的 client 将能够在根据旧接口编译的 server 上调用 add()(server 将假设第二个参数为零)。
//
// * 根据此接口编译的 server 将能够接受根据旧接口编译的 client 的 add() 调用(client 的第二个参数将被忽略)。
RCF_BEGIN(I_Calculator, "I_Calculator")
    RCF_METHOD_R1(double, add, double)
RCF_END(I_Calculator)
```
注意，RCF 封送输入参数和输出参数的顺序与它们在 `RCF_METHOD_XX()` 声明中出现的顺序相同。任何添加(或删除)的参数必须是最后编组的，否则将破坏兼容性。

#### 1.3 重命名接口
RCF 接口由它们的运行时名称标识，在 `RCF_BEGIN()` 宏的第二个参数中指定。只要保留这个名称，就可以更改接口的编译时名称，而不会破坏兼容性。
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
### 2. 归档版本控制
在远程调用中传递的应用程序特定的数据类型可能会随着时间而改变。随着这些数据类型的变化，它们的序列化函数也会发生变化。为了帮助应用程序维护序列化代码的向后兼容性，RCF 提供了一个归档版本号概念。

归档版本号在每次远程调用时都从 client 传递到 server，缺省情况下为零。要实现版本控制，您的应用程序需要管理归档版本，本质上是在对序列化代码进行中断更改时更新它。

要设置归档版本号，请在创建任何 server 或 client 之前调用 [RCF::setArchiveVersion()](http://www.deltavsoft.com/doc/_version_8hpp.html#a93f3cfe224b97643d9de169f4c28ba52)。

或者，您可以通过调用 [RCF::ClientStub::setArchiveVersion()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a6d5c0464fb9038da43e512410385ef40) 和 [RCF::RcfServer::setArchiveVersion()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#a5c7483f68d8b378095ecc2122b31edd3) 来为单个 client 和 server 设置归档版本号。

当 client 连接到 server 时，会自动执行版本协商(version negotiation)步骤，以确保连接使用两个组件都支持的最大归档版本号。

归档版本号用于序列化代码。通过调用 [SF::Archive::getArchiveVersion()](http://www.deltavsoft.com/doc/class_s_f_1_1_archive.html#ab57895175bdee4ff0b1e2018a06c520b)，可以从一个序列化函数中检索正在使用的归档版本号：
```cpp
void serialize(SF::Archive & ar, MyClass & m){
    std::uint32_t archiveVersion = ar.getArchiveVersion();
    // ...
}
```
然后，序列化代码可以使用归档版本号的值来确定要序列化哪些成员。

例如，假设应用程序的第一个版本包含以下代码：
```cpp
class X{
public:
    int mA = 0;
    void serialize(SF::Archive & ar){
        ar & mA;
    }
};
RCF_BEGIN(I_Echo, "I_Echo")
    RCF_METHOD_R1(X, Echo, const X &)
RCF_END(I_Echo)
class EchoImpl{
public:
    X Echo(const X & x) { 
        return x; 
    }
};
```
```cpp
    //--------------------------------------------------------------------------
    // 接受来自其他进程的调用…
    EchoImpl echo;
    RCF::RcfServer server( RCF::TcpEndpoint(50001) );
    server.bind<I_Echo>(echo);
    server.start();
    //--------------------------------------------------------------------------
    // ... 或调用其他进程。
    RcfClient<I_Echo> client( RCF::TcpEndpoint(50002) );
    X x1;
    X x2;
    x2 = client.Echo(x1);
```
一旦这个版本发布，一个新的版本就准备好了，`X` 类中添加了一个新成员：
```cpp
class X{
public:
    int mA = 0;
    int mB = 0;
    void serialize(SF::Archive & ar){
        // 检索归档版本，以确定要序列化哪些成员。
        std::uint32_t version = ar.getArchiveVersion();
        ar & mA;
        if (version >= 1) {
            ar & mB;
        }
    }
};
class EchoImpl{
public:
    X Echo(const X & x) { return x; }
};
```
注意，`X` 的序列化代码现在使用归档版本号来确定是否应该序列化新的 `mB` 成员。

通过这些更改，新 server 可以处理来自旧 client 和新 client 的调用，新 client 可以调用旧 server 或新 server ：
```cpp
    // 指定的归档版本应该是此进程支持的最新归档版本。
    RCF::setArchiveVersion(1);
    //--------------------------------------------------------------------------
    // 接受来自其他进程的调用…
    // 这个 server 可以接受新旧 client 的调用。归档版本在旧 client 调用时为0，在新 client 调用时为1。
    EchoImpl echo;
    RCF::RcfServer server( RCF::TcpEndpoint(50001) );
    server.bind<I_Echo>(echo);
    server.start();
    //--------------------------------------------------------------------------
    // ... or making calls to other processes.
    // This client can call either new or old servers.
    RcfClient<I_Echo> client( RCF::TcpEndpoint(50002) );
    X x1;
    X x2;
    // If the server on port 50002 is old, this call will have archive version set to 0.
    // If the server on port 50002 is new, this call will have archive version set to 1.
    x2 = client.Echo(x1);
```
### 3. 运行时版本控制
RCF 保持与自身的运行时兼容性，适用于 RCF 2.0 之前的版本(包括RCF 2.0)。不保证运行时与大于 2.0 的 RCF 版本兼容。

为了实现 RCF 版本之间的运行时兼容性，RCF 维护一个运行时版本号，每个 RCF 版本的运行时版本号都是递增的。运行时版本号在每个远程调用的请求头中传递，并允许旧的和新的 RCF 版本互操作。

RCF 的自动 client-server 版本协商处理运行时版本控制和归档版本控制。在大多数情况下，您不需要知道运行时版本号。您可以混合使用和匹配 RCF 版本，并且在运行时，为每个 client-server 连接协商合适的运行时版本。

#### 3.1 自定义版本协商
在某些情况下，client 可能已经知道它将要调用的 server 的运行时版本和归档版本，在这种情况下，它可以禁用自动版本控制，并显式地设置版本号：
```cpp
    RcfClient<I_Echo> client( RCF::TcpEndpoint(50001) );
    // 关闭自动版本协商。
    client.getClientStub().setAutoVersioning(false);
    // client 显式地设置版本号以匹配 RCF 2.0 和归档版本 2。
    client.getClientStub().setRuntimeVersion(10); // RCF 2.0
    client.getClientStub().setArchiveVersion(2);
    // 如果 server 不支持请求的版本号，将抛出异常。
    X x1;
    X x2 = client.Echo(x1);
```
不支持单向调用的自动版本协商。特别是，发布者向其订阅者发出的单向调用不会自动进行版本控制。如果您的订阅者具有不同的运行时和归档版本号，发布者将需要显式地在发布 `RcfClient<>` 对象上设置版本号，以匹配最旧的预期订阅者：
```cpp
    RCF::RcfServer server( RCF::TcpEndpoint(50001) );
    server.start();
    
    typedef RCF::Publisher<I_PrintService> PrintServicePublisher;
    typedef std::shared_ptr< PrintServicePublisher > PrintServicePublisherPtr;
    PrintServicePublisherPtr pubPtr = server.createPublisher<I_PrintService>();
    // client 显式设置版本号以支持较旧的订阅者。
    pubPtr->publish().getClientStub().setRuntimeVersion(10); // RCF runtime version 10 (RCF 2.0).
    pubPtr->publish().getClientStub().setArchiveVersion(5); // Application archive version 5.
```
作为参考，下面是每个 RCF 版本的运行时版本号表。
|RCF 发布版 |   运行时版本号|
|--|--|
|2.0|	10|
|2.1|	11|
|2.2|	12|
|3.0|	13|
### 4. 协议缓冲区(Protocol Buffers)
对于具有向后兼容性要求和较短或连续的发布周期的应用程序，归档版本控制可能变得难以管理。归档版本号的每个增量都涉及向序列化函数添加新的执行路径，并且随着时间的推移可能会导致复杂的序列化代码。

RCF 还支持[协议缓冲区](https://developers.google.com/protocol-buffers/)，它提供了另一种版本控制方法。协议缓冲区可以用来生成带有内置序列化代码的 C++ 类，而不是手工为 C++ 对象编写序列化代码，这些代码可以自动处理版本差异（ 请参阅[协议缓冲区](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/external_serialization.md) ）。