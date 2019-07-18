<!--
 * @Author: haoluo
 * @Date: 2019-07-15 16:34:18
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-18 15:58:38
 * @Description: file content
 -->
## 序列化
序列化是每个远程调用的基本部分。远程调用参数需要序列化为二进制格式，以便在网络上传输，一旦接收到，需要将其从二进制格式反序列化回常规 C++ 对象。

RCF 提供了一个内置的序列化框架，它可以自动处理常见 C++ 类型的序列化，并且可以自定义来序列化任意用户定义的 C++ 类型。
### 1. 标准 C++ 类型
基本 C++ 类型(`char`、`int`、`double`等)的序列化由 RCF 自动处理。序列化其他一些常见和标准的 C++ 类型需要包含相关的头文件：
|类型  |要包含的序列化头|
|--|--|
|std::string, std::wstring, std::basic_string<>	|`#include <SF/string.hpp>`|
|std::vector<>|	`#include <SF/vector.hpp>`|
|std::list<>|	`#include <SF/list.hpp>`|
|std::deque<>|	`#include <SF/deque.hpp>`|
|std::set<>|	`#include <SF/set.hpp>`|
|std::map<>|	`#include <SF/map.hpp>`|
|std::pair<>|	`#include <SF/utility.hpp>`|
|std::bitset<>|	`#include <SF/bitset.hpp>`|
|std::auto_ptr<>|	`#include <SF/auto_ptr.hpp>`|
|std::unique_ptr<>|	`#include <SF/unique_ptr.hpp>`|
|std::shared_ptr<>|	`#include <SF/shared_ptr.hpp>`|
|std::tuple<>|	`#include <SF/tuple.hpp>`|
|std::array<>|	`#include <SF/array.hpp>`|
|boost::scoped_ptr<>|	`#include <SF/boost/scoped_ptr.hpp>`|
|boost::shared_ptr<>|	`#include <SF/boost/shared_ptr.hpp>`|
|boost::intrusive_ptr<>|	`#include <SF/boost/intrusive_ptr.hpp>`|
|boost::any|	`#include <SF/boost/any.hpp>`|
|boost::tuple<>|	`#include <SF/boost/tuple.hpp>`|
|boost::variant<>|	`#include <SF/boost/variant.hpp>`|
|boost::array<>|	`#include <SF/boost/array.hpp>`|
C++ enum 被自动序列化为整数。C++11 enum 类的序列化需要使用一个 helper 宏：
```cpp
// Legacy C++ enum，自动序列化为'int'
enum Suit{
    Heart = 1,
    Diamond = 2,
    Club = 2,
    Spade = 2
};
// 具有自定义基类型(8位整数)的 C++11 enum 类。
enum class Colors : std::int8_t { 
    Red = 1, 
    Green = 2, 
    Blue = 3 
};
// 使用 SF_SERIALIZE_ENUM_CLASS() 指定 enum 类的基类型。
SF_SERIALIZE_ENUM_CLASS(Colors, std::int8_t)
```

### 2. 用户定义的类型
如果你有一个自己的类，并在 RCF 接口中使用它：
```cpp
class Point3D{
public:
    double mX;
    double mY;
    double mZ;
};
RCF_BEGIN(I_Echo, "I_Echo")
    RCF_METHOD_R1(Point3D, Echo, const Point3D &)
RCF_END(I_Echo)
```
，你会得到一个类似于下面这样的编译器错误：
```shell
..\\..\\..\\..\\..\include\SF\Serializer.hpp(324) : error C2039: 'serialize' : is not a member of 'Point3D'
        C6.cpp(13) : see declaration of 'Point3D'
        ..\\..\\..\\..\\..\include\SF\Serializer.hpp(336) : see reference to function template instantiation 'void SF::serializeInternal<T>(SF::Archive &,T &)' being compiled
        with
        \[
            T=U
        \]
        <snip>
```
编译器告诉我们，它找不到类 `Point3D` 的任何序列化代码。我们需要提供一个 `serialize()` 函数，要么作为一个成员函数：
```cpp
class Point3D{
public:
    double mX;
    double mY;
    double mZ;
    // 内部序列化
    void serialize(SF::Archive &ar){
        ar & mX & mY & mZ;
    }
};
```
，要么作为与 Point3D 相同的命名空间中的一个自由函数，或作为 SF 命名空间中的一个自由函数：
```cpp
// 外部序列化
void serialize(SF::Archive &ar, Point3D &point){
    ar & point.mX & point.mY & point.mY;
}
```
`serialize()` 函数中的代码指定要序列化哪些成员。

`serialize()` 函数的作用是：序列化和反序列化。在某些情况下，您可能希望使用不同的逻辑，这取决于代码是序列化还是反序列化一个对象。例如，下面的代码片段实现了 `boost::gregorian::date` 类的序列化，通过将其表示为一个字符串：
```cpp
// boost::gregorian::date 的序列化代码可以放在 boost::gregorian::date 命名空间(其中定义了path类)中，
// 也可以放在 SF 命名空间中。这里我们选择了 SF 命名空间。
namespace SF {
    void serialize(SF::Archive & ar, boost::gregorian::date & dt){
        if (ar.isWrite()) {
            // Code for serializing.
            std::ostringstream os;
            os << dt;
            std::string s = os.str();
            ar & s;
        } else {
            // Code for deserializing.
            std::string s;
            ar & s;
            std::istringstream is(s);
            is >> dt;
        }
    }
}
```

### 3. 二进制数据
要发送二进制数据块，可以使用 [RCF::ByteBuffer](http://www.deltavsoft.com/doc/class_r_c_f_1_1_byte_buffer.html) 类：
```cpp
RCF_BEGIN(I_Echo, "I_Echo")
    RCF_METHOD_R1(RCF::ByteBuffer, Echo, RCF::ByteBuffer)
RCF_END(I_Echo)
class EchoImpl{
public:
    RCF::ByteBuffer Echo(RCF::ByteBuffer byteBuffer){
        return byteBuffer;
    }
};
} 
int main(){
    EchoImpl echoImpl;
    RCF::RcfServer server( RCF::TcpEndpoint(0));
    server.bind<I_Echo>(echoImpl);
    server.start();
    int port = server.getIpServerTransport().getPort();
    // Create and fill a 500 kb byte buffer.
    RCF::ByteBuffer byteBuffer(500*1024);
    for (std::size_t i=0; i<byteBuffer.getLength(); ++i) {
        byteBuffer.getPtr()[i] = char(i % 256);
    }
    RcfClient<I_Echo> client(( RCF::TcpEndpoint(port) ));
    // Echo it.
    RCF::ByteBuffer byteBuffer2 = client.Echo(byteBuffer);
    return 0;
}
```
`std::string` 或 `std::vector<char>` 可以用于相同的目的。但是，[RCF::ByteBuffer](http://www.deltavsoft.com/doc/class_r_c_f_1_1_byte_buffer.html) 的序列化和封装处理效率要高得多（ 请参阅[性能](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/performance.md) ）。使用 `RCF::ByteBuffer`，在连接的两端都不会对数据进行任何复制。

### 4. 可移植性
因为 C++ 没有规定其基本类型的大小，所以当 server 和 client 部署在不同的平台上时，可能会出现序列化错误。例如，考虑以下接口：
```cpp
RCF_BEGIN(I_Echo, "I_Echo")
    RCF_METHOD_R1(std::size_t, Echo, std::size_t)
RCF_END(I_Echo)
```
`Echo()` 方法使用 `std::size_t` 作为参数和返回值。

不幸的是，`std::size_t` 在不同的平台上有不同的含义。例如，32 位 Visual C++ 编译器认为 `std::size_t` 是 32 位类型，而 64 位 Visual C++ 编译器认为 `std::size_t` 是 64 位类型。因此，如果使用带有 `std::size_t` 参数的方法从 32 位 client 对 64 位 server 进行远程调用，将引发运行时序列化错误。

当在 32 位和 64 位版本的 gcc 中使用 `long` 类型时，也会出现同样的问题。32 位 gcc 认为 `long` 是 32 位类型，而 64 位 gcc 认为 `long` 是 64 位类型。

在这些情况下，正确的方法是使用 `typedefs` 来确保算术类型的位大小。特别是，标准 `C++ <cstdint>` 头文件提供了许多有用的 `typedefs`，包括以下内容。

|类型|   描述|
|--|--|
|std::int16_t|	16 位有符号整数|
|std::uint16_t|	16 位无符号整数|
|std::int32_t|	32 位有符号整数|
|std::uint32_t|	32 位无符号整数|
|std::int64_t|	64 位有符号整数|
|std::uint64_t|	64 位无符号整数|

为了确保可移植性，上面的 `std::size_t` 示例应该重写为：
```cpp
RCF_BEGIN(I_Echo, "I_Echo")
    RCF_METHOD_R1(std::uint32_t, Echo, std::uint32_t)
    RCF_METHOD_R1(std::uint64_t, Echo, std::uint64_t)
RCF_END(I_Echo)
```

### 5. 从磁盘和到磁盘的序列化
RCF 序列化可用于在磁盘和磁盘之间序列化对象。为此，使用 [SF::OBinaryStream](http://www.deltavsoft.com/doc/class_s_f_1_1_o_binary_stream.html) 和 [SF::IBinaryStream](http://www.deltavsoft.com/doc/class_s_f_1_1_i_binary_stream.html)：
```cpp
std::string filename = "data.bin";
X x1;
X x2;
// Write x1 to a file.
{
    std::ofstream fout(filename.c_str(), std::ios::binary);
    SF::OBinaryStream os(fout);
    os << x1;
}
// Read x2 from a file.
{
    
    std::ifstream fin(filename.c_str(), std::ios::binary);
    SF::IBinaryStream is(fin);
    is >> x2;
}
```
有关序列化的更高级主题，请参阅[高级序列化](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/advanced_serialization.md)。