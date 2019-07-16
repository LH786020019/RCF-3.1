<!--
 * @Author: haoluo
 * @Date: 2019-07-16 10:32:01
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-16 10:34:10
 * @Description: file content
 -->
## 外部序列化
协议缓冲区
协议缓冲区是谷歌发布的一个开源项目，它使用一种专门的语言来定义序列化模式，并从中生成实际的序列化代码。官方协议缓冲区编译器可以生成多种语言的代码，包括c++、c#、Java和Python。

协议缓冲区类
RCF为协议缓冲区编译器生成的类提供了本地封送处理支持。要启用此支持，在构建RCF时定义RCF_FEATURE_PROTOBUF=1。定义了RCF_USE_PROTOBUF=1后，协议缓冲区编译器生成的类可以直接在RCF接口声明中使用，并将使用协议缓冲区运行时进行序列化和反序列化。

有关使用协议缓冲区支持构建RCF的更多信息，请参见构建RCF。

例如，考虑这个.proto文件。
```proto
// Person.proto
message Person {
  int32 id = 1;
  string name = 2;
  string email = 3;
}
```
在Person上运行协议缓冲区编译器(在c++模式下)。proto生成Person.pb和Person.pb这两个文件。cc，包含c++类Person的实现。
```cpp
// Person.pb.h
class Person : public ::google::protobuf::Message {
// ...
}
```
现在，您可以在程序中包含Person.pb.h，并在RCF接口中使用class Person。RCF将检测Person类是否是协议缓冲区类，并使用协议缓冲区函数来序列化和反序列化Person实例。
```cpp
#include <RCF/../../test/protobuf/messages/cpp/Person.pb.h>
namespace { 
RCF_BEGIN(I_X, "I_X")
    RCF_METHOD_R1(Person, Echo, const Person &)
RCF_END(I_X)
```
协议缓冲区类可以与原生c++类一起使用，在RCF接口中：
```cpp
    RCF_BEGIN(I_X, "I_X")
        RCF_METHOD_R1(Person, Echo, const Person &)
        RCF_METHOD_V4(void , SomeOtherMethod, int, const std::vector<std::string> &, Person &, RCF::ByteBuffer)
    RCF_END(I_X)
```
协议缓冲区高速缓存
协议缓冲区生成的序列化和反序列化代码经过了高度优化。然而，为了充分利用这种性能，您可能希望在调用之间缓存和重用协议缓冲区对象，而不是创建新的对象。协议缓冲区对象的设计目的是保留它们分配的任何内存，并将在后续的序列化和反序列化操作中重用这些内存。

为此，RCF提供了服务器端对象缓存。下面是一个启用Person对象的服务器端缓存的例子：
```cpp
    RCF::ObjectPool & cache = RCF::getObjectPool();
    // Enable server-side caching for Person.
    // * Don't cache more than 10 Person objects.
    // * Call Person::Clear() before putting a Person object into the cache.
    auto clearPerson = [](Person * pPerson) { pPerson->Clear();  };
    cache.enableCaching<Person>(10, clearPerson);
```
Boost.Serialization
过去版本的RCF支持Boost的使用。序列化，作为RCF内部序列化框架的替代。

然而，从RCF 3.0开始，Boost的使用就出现了。序列化是弃用。

支持提升。序列化将在将来的版本中从代码库中删除。