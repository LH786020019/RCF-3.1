<!--
 * @Author: haoluo
 * @Date: 2019-07-12 14:16:08
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-12 14:16:08
 * @Description: file content
 -->
### RCF 是什么？
`RCF` (Remote Call Framework，远程调用框架)是一个面向 C++ 的跨平台进程间通信框架。

与其他通信框架不同，RCF 不使用单独的 IDL(Interface Definition Language，接口定义语言)。RCF 接口是在 C++ 中直接定义的，用户定义数据类型的序列化也是在 C++ 中实现的。代替一个单独的 IDL 编译器工具，RCF 使用 C++ 编译器生成 `client` 和 `server` 存根。

RCF提供了广泛的进程间通信特性：
- Oneway 和 twoway 消息传递。
- Batched oneway 消息传递。
- 发布/订阅（Publish/subscribe）风格的消息传递。
-  通过UDP多播（Multicast）和广播（Broadcast）消息传递。
- 异步远程调用（Asynchronous remote calls）。
- Server-to-client 回调连接。
- 多种传输方式（TCP、UDP、Windows 命名管道和 UNIX 本地域套接字）。
- 通过 HTTP 和 HTTPS 进行 tunnel。
- 压缩（zlib）和加密（Kerberos、NTLM、Schannel和OpenSSL）。
- 支持 IPv6。
- 内置序列化框架。
- 内置文件传输功能。
- 健壮的版本支持。
- 支持[Boost.Serialization](https://blog.csdn.net/qq_23599965/article/details/89375475#1_BoostSerialization_2)。
- 支持[Protocol Buffers](https://blog.csdn.net/qq_23599965/article/details/89375475#2_Protocol_Buffers_28)。
- 支持JSON-RPC。
- 可移植性，适用于多种编译器、平台和操作系统。
- 可伸缩性，适用于广泛的应用程序，从简单的parent-child IPC，到大型分布式系统。
- 效率，在服务器和客户机上的关键路径上实现零拷贝（zero-copy）、零堆分配（zero-heap allocation）。
- 没有依赖，除了[Boost](https://www.boost.org/)头文件(1.35.0或更高版本)。[zlib](http://www.zlib.net/)和[OpenSSL](http://www.openssl.org/)是可选的。

要开始学习RCF编程，请直接进入教程。
