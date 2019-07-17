<!--
 * @Author: haoluo
 * @Date: 2019-07-16 10:32:01
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-17 18:42:49
 * @Description: file content
 -->
## 外部序列化
### 1. 协议缓冲区(Protocol Buffers)
[协议缓冲区](https://github.com/google/protobuf)是 [Google](http://www.google.com/) 发布的一个开源项目，它使用一种专门的语言来定义序列化模式，并从中生成实际的序列化代码。官方的协议缓冲区编译器可以生成多种语言编写的代码，包括 `C++`、`C#`、`Java` 和 `Python`。

#### 1.1 协议缓冲区类(Protocol Buffer Classes)
RCF 为协议缓冲区编译器生成的类提供了本地封装支持。要启用此支持，在构建 RCF 时定义 `RCF_FEATURE_PROTOBUF=1`。定义了 `RCF_USE_PROTOBUF=1` 后，协议缓冲区编译器生成的类可以直接在 RCF 接口声明中使用，并将使用协议缓冲区运行时进行序列化和反序列化。

有关使用协议缓冲区支持构建 RCF 的更多信息，请参阅[构建 RCF](https://love2.io/@lh786020019/doc/RCF-3.1/building_RCF/index.md)。

例如，考虑这个 `.proto` 文件。
```proto
// Person.proto
message Person {
  int32 id = 1;
  string name = 2;
  string email = 3;
}
```
在 `Person.proto` 上运行协议缓冲区编译器(在C++模式下)将会生成 `Person.pb.h` 和 `Person.pb.cc` 这两个文件，它们包含 C++ 类 `Person` 的实现。
```cpp
// Person.pb.h
class Person : public ::google::protobuf::Message {
// ...
}
```
现在，您可以在程序中包含 `Person.pb.h`，并在 RCF 接口中使用类 `Person`。RCF 将检测 `Person` 类是否是协议缓冲区类，并使用协议缓冲区函数来序列化和反序列化 `Person` 实例。
```cpp
#include <RCF/../../test/protobuf/messages/cpp/Person.pb.h>
namespace { 
RCF_BEGIN(I_X, "I_X")
    RCF_METHOD_R1(Person, Echo, const Person &)
RCF_END(I_X)
```
协议缓冲区类可以与原生 C++ 类一起使用，在 RCF 接口中：
```cpp
    RCF_BEGIN(I_X, "I_X")
        RCF_METHOD_R1(Person, Echo, const Person &)
        RCF_METHOD_V4(void , SomeOtherMethod, int, const std::vector<std::string> &, Person &, RCF::ByteBuffer)
    RCF_END(I_X)
```

#### 1.2 协议缓冲区缓存(Protocol Buffer Caching)
协议缓冲区生成的序列化和反序列化代码经过了高度优化。然而，为了充分利用这种性能，您可能希望在调用之间缓存和重用协议缓冲区对象，而不是创建新的对象。协议缓冲区对象的设计目的是保留它们分配的任何内存，并将在后续的序列化和反序列化操作中重用这些内存。

为此，RCF 提供了 [Server 端对象缓存](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/performance.md)。下面是一个启用 `Person` 对象的 server 端缓存的例子：
```cpp
    RCF::ObjectPool & cache = RCF::getObjectPool();
    // 为 Person 启用 server 端缓存。
    // * 不要缓存超过 10 个 Person 对象。
    // * 在将一个 Person 对象放入缓存之前调用 `Person::Clear()`。
    auto clearPerson = [](Person * pPerson) { pPerson->Clear();  };
    cache.enableCaching<Person>(10, clearPerson);
```

### 2. Boost.Serialization
过去版本的 RCF 支持 [Boost.Serialization](http://www.boost.org/libs/serialization) 的使用，以作为 RCF 内部序列化框架的替代。

**然而，从 RCF 3.0 开始，Boost.Serialization 的使用就被弃用了。**

`Boost.Serialization` 的支持将在将来的版本中从代码库中删除。