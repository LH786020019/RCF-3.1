<!--
 * @Author: haoluo
 * @Date: 2019-07-12 16:02:23
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-18 16:14:24
 * @Description: file content
 -->
# 构建 RCF
RCF 作为源代码提供，并被设计为与包含您的应用程序的其余源代码一起编译。要编译 RCF，您只需编译一个文件 ——— `RCF.cpp`，位于 `src/RCF/RCF.cpp` 的发行版中。

您可以直接将 RCF.cpp 编译到与您自己的源代码相同的模块中，也可以将其编译成静态或动态库，然后将其链接到您的应用程序中。

RCF 对任何其他库没有任何强制依赖关系。以前的 RCF 版本(2.2或更早版本)要求 [Boost](http://www.boost.org/) 头文件可用，但是从 RCF 3.0 开始，情况就不一样了。

RCF 有几个可选的依赖项([zlib](http://www.zlib.net/)、[OpenSSL](http://www.openssl.org/)、[Protocol Buffers](http://code.google.com/p/protobuf/))，这些依赖项是通过下面列出的特性定义启用的。

假设您已经将 RCF 下载到计算机上的目录 `<rcf_distro>`。

要构建 RCF，请遵循以下步骤：
- 定义任何相关的 RCF 构建或特性定义。
- 将 `<rcf_distro>/include` 设置为一个 include 路径。
- 编译 `<rcf_distro>/src/RCF/RCF.cpp`。
- 链接到任何相关的第三方库。

### 1. 构建示例
编译和链接的实际细节将根据您使用的特定编译器而有所不同。
- 如果使用 [Visual Studio](http://http//www.microsoft.com/visualstudio)，需要在 Visual C++ 项目的项目属性中设置 include 路径、编译器定义、链接器路径和链接器输入。
- 如果使用命令行工具(如 [gcc](http://http//gcc.gnu.org/) 或 [clang](https://clang.llvm.org/)，带有一些 makefiles 的味道)，则需要在相关命令行上设置 include 路径、编译器定义、链接器路径和链接器输入。
作为一个示例，让我们从示例代码章节构建最小的 TCP client 和 server。首先，创建一个名为 `SampleTcpClientServer.cpp` 的源文件，并将示例代码复制粘贴到其中。然后按照以下指示：

#### 1.1 Visual Studio
- 在 Visual Studio IDE 中，选择 `New Project -> Visual C++ -> Win32 -> Win32 Console Application`
- 在`应用程序设置(Application Settings)`对话框中，选中`空项目(Empty Project)`复选框。
- 在新建项目的`项目属性(Project Properties)`对话框中，选择 `C/C++ -> General -> Additional Include Directories`。
- 将 include 路径添加到RCF - `<rcf_distro>\include`。
- 将 `SampleTcpClientServer.cpp` 文件添加到项目中。
- 将 `<rcf_distro>\src\RCF\RCF.cpp` 文件添加到项目中。
- 选择 `Build -> Build Solution`。

#### 1.2 gcc
从与 `SampleTcpClientServer.cpp` 相同的目录，运行以下命令：
```shell
g++ -std=c++1y SampleTcpClientServer.cpp ../../src/RCF/RCF.cpp -I../../include -lpthread -luuid -ldl -oSampleTcpClientServer
```

### 2. 动态和静态链接
上面的示例将 `RCF.cpp` 直接编译到与 `SampleTcpClientServer.cpp` 相同的 C++ 模块中。我们还可以将 `RCF.cpp` 编译成一个静态或动态库。
- 要构建一个静态库，请设置用于创建一个静态库的相关的编译器开关，并像往常一样编译 RCF.cpp。
- 要构建一个动态库，请在 `build` 中定义 `RCF_BUILD_DLL`，设置用于创建一个动态库的任何相关的编译器开关，然后编译 RCF.cpp。

### 3. 构建定义
下表列出了 RCF 构建定义。

| Build define |	意义 |
| -- | -- |
|RCF_BUILD_DLL|	构建一个 DLL 导出 RCF。|
|RCF_USE_CUSTOM_ALLOCATOR| 使用自定义内存分配器支持构建。	|
|RCF_USE_CLOCK_MONOTONIC|	使用 `clock_gettime()` 实现一个单调时钟(monotonic clock)。|
|RCF_MAX_METHOD_COUNT=<N>|	扩展一个 RCF 接口中允许的最大方法数量。|

#### 3.1 笔记
- `RCF_BUILD_DLL`
在构建动态库时需要定义 RCF_BUILD_DLL。它将 public RCF 类和函数标记为 `DLL` 导出。
- `RCF_USE_CUSTOM_ALLOCATOR`
RCF_USE_CUSTOM_ALLOCATOR 用于为网络发送和接收缓冲区自定义内存分配。如果定义了 RCF_USE_CUSTOM_ALLOCATOR，则需要实现以下两个函数：
```cpp
namespace RCF {
    void * RCF_new(std::size_t bytes);
    void RCF_delete(void * ptr);
} // namespace RCF
```
每当为发送或接收的数据缓冲区分配或释放内存时，RCF 将调用 `RCF_new()` 和 `RCF_delete()`。
- `RCF_USE_CLOCK_MONOTONIC`
当 RCF_USE_CLOCK_MONOTONIC 被定义时，RCF 将使用 `clock_gettime()` 和 `CLOCK_MONOTONIC` 来实现内部时钟功能。否则，RCF 在 POSIX 平台上使用 `gettimefoday()`，在 Windows 平台上使用 `GetTickCount()`。
- `RCF_MAX_METHOD_COUNT=<N>`
默认情况下，RCF 接口中允许的最大方法数是100。可以通过将 RCF_MAX_METHOD_COUNT 设置为自定义值来更改此限制。最大允许值为200。

### 4. 特性定义
RCF 附带一组特性定义，允许用户使用一组特定的特性构建 RCF。

下表列出了所有可用的特性定义。

|Feature define|	RCF feature|	Default value|
| -- | -- | -- |
|RCF_FEATURE_SSPI|	 Win32基于SSPI的加密(NTLM, Kerberos, Schannel)	|1 on Windows. 0 otherwise.|
|RCF_FEATURE_FILETRANSFER	|	文件传输|0|
|RCF_FEATURE_SERVER|	非关键的 server 端功能(server到client的ping回调、server对象、会话超时)|	1|
|RCF_FEATURE_PROXYENDPOINT|	代理端点|	1|
|RCF_FEATURE_PUBSUB|	Publish/subscribe|	1|
|RCF_FEATURE_TCP|	TCP传输|	1|
|RCF_FEATURE_UDP|	UDP传输	|1|
|RCF_FEATURE_NAMEDPIPE|	Win32命名管道传输	|1 on Windows. 0 otherwise.|
|RCF_FEATURE_LOCALSOCKET|	Unix本地套接字传输	|0 on Windows. 1 otherwise.|
|RCF_FEATURE_HTTP|	HTTP/HTTPS传输	|1|
|RCF_FEATURE_IPV6|	IPv6支持	|1|
|RCF_FEATURE_SF|	RCF内部序列化	|1|
|RCF_FEATURE_LEGACY|	与RCF 1.x和更早版本的向后兼容性	|0|
|RCF_FEATURE_ZLIB| 基于Zlib的压缩	|	0|
|RCF_FEATURE_OPENSSL|	基于OpenSSL的SSL加密	|0|
|RCF_FEATURE_PROTOBUF|	Protocol Buffer支持	|0|

要使用这些定义，请将它们设置为0或1，这取决于是否应该包含该特性。

例如，要构建不支持 `HTTP/HTTPS` 的 RCF，请在您的 build 中定义 `RCF_FEATURE_HTTP=0`。

#### 4.1笔记
- `RCF_FEATURE_FILETRANSFER`
此文件传输特性使用标准 `C++ <filesystem>` 库。此库应该在任何支持 C++17 语言更新的编译器上可用。注意，如果您正在使用 gcc 或 clang，可能需要使用 `-std=c++17` 编译器选项来启用 cC++17 支持。
- `RCF_FEATURE_ZLIB`
如果您想通过定义 `RCF_FEATURE_ZLIB=1` 启用此特性，则需要链接到 `Zlib`。可以通过定义 `RCF_ZLIB_STATIC` 静态链接到 Zlib。
- `RCF_FEATURE_OPENSSL`
如果您想通过定义 `RCF_FEATURE_OPENSSL=1` 启用此特性，则需要链接到 `OpenSSL`。通过定义 `RCF_OPENSSL_STATIC`，可以静态地链接到 OpenSSL。
- `RCF_FEATURE_PROTOBUF`
如果您想通过定义 `RCF_FEATURE_PROTOBUF=1` 启用此特性，则需要链接到 `Protocol Buffers`。

### 5. 支持的编译器
RCF 是用标准 C++ 编写的，因此应该构建在任何最新的符合标准的编译器之上。RCF 3.0 已在以下编译器上测试：
- Visual C++ 2015(x86和x64)
- Visual C++ 2017(x86和x64)
- gcc 7.0(x64)
- clang 4.0(x64)

### 6. 支持的平台
RCF 已经在 Windows 和 Linux 平台上测试过，但是您应该能够在以下任何一个平台上使用它：
- Windows
- Linux
- OS X
- Solaris
- FreeBSD