<!--
 * @Author: haoluo
 * @Date: 2019-07-15 16:34:18
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-15 16:47:05
 * @Description: file content
 -->
序列化
序列化是每个远程调用的基本部分。远程调用参数需要序列化为二进制格式，以便在网络上传输，一旦接收到，需要将其从二进制格式反序列化回常规c++对象。

RCF提供了一个内置的序列化框架，它可以自动处理常见c++类型的序列化，并且可以定制来序列化任意用户定义的c++类型。
标准c++类型
基本c++类型(char、int、double等)的序列化由RCF自动处理。序列化其他一些常见和标准的c++类型需要包含相关的头文件：
类型  要包含的序列化头
std::string, std::wstring, std::basic_string<>	#include <SF/string.hpp>
std::vector<>	#include <SF/vector.hpp>
std::list<>	#include <SF/list.hpp>
std::deque<>	#include <SF/deque.hpp>
std::set<>	#include <SF/set.hpp>
std::map<>	#include <SF/map.hpp>
std::pair<>	#include <SF/utility.hpp>
std::bitset<>	#include <SF/bitset.hpp>
std::auto_ptr<>	#include <SF/auto_ptr.hpp>
std::unique_ptr<>	#include <SF/unique_ptr.hpp>
std::shared_ptr<>	#include <SF/shared_ptr.hpp>
std::tuple<>	#include <SF/tuple.hpp>
std::array<>	#include <SF/array.hpp>
boost::scoped_ptr<>	#include <SF/boost/scoped_ptr.hpp>
boost::shared_ptr<>	#include <SF/boost/shared_ptr.hpp>
boost::intrusive_ptr<>	#include <SF/boost/intrusive_ptr.hpp>
boost::any	#include <SF/boost/any.hpp>
boost::tuple<>	#include <SF/boost/tuple.hpp>
boost::variant<>	#include <SF/boost/variant.hpp>
boost::array<>	#include <SF/boost/array.hpp>
c++枚举被自动序列化为整数。c++ 11枚举类的序列化需要使用helper宏：
```cpp
// Legacy C++ enum. Automatically serialized as 'int'.
enum Suit
{
    Heart = 1,
    Diamond = 2,
    Club = 2,
    Spade = 2
};
// C++11 enum class with custom base type (8 bit integer).
enum class Colors : std::int8_t 
{ 
    Red = 1, 
    Green = 2, 
    Blue = 3 
};
// Use SF_SERIALIZE_ENUM_CLASS() to specify the base type of the enum class.
SF_SERIALIZE_ENUM_CLASS(Colors, std::int8_t)
```
用户定义的类型
如果你有一个自己的类，并在RCF接口中使用它：
```cpp
class Point3D
{
public:
    double mX;
    double mY;
    double mZ;
};
RCF_BEGIN(I_Echo, "I_Echo")
    RCF_METHOD_R1(Point3D, Echo, const Point3D &)
RCF_END(I_Echo)
```
，你会得到一个类似于这个编译器错误：
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
编译器告诉我们，它找不到类Point3D的任何序列化代码。我们需要提供一个serialize()函数，要么作为成员函数：
```cpp
class Point3D
{
public:
    double mX;
    double mY;
    double mZ;
    // Internal serialization.
    void serialize(SF::Archive &ar)
    {
        ar & mX & mY & mZ;
    }
};
```
，或作为与Point3D相同的名称空间中的自由函数，或作为SF名称空间中的自由函数：
```cpp
// External serialization.
void serialize(SF::Archive &ar, Point3D &point)
{
    ar & point.mX & point.mY & point.mY;
}
```
函数中的代码指定要序列化哪些成员。

函数的作用是:序列化和反序列化。在某些情况下，您可能希望使用不同的逻辑，这取决于代码是序列化还是反序列化对象。例如，下面的代码片段实现了boost::gregorian::date类的序列化，将其表示为字符串：
```cpp
// boost::gregorian::date的序列化代码可以放在boost::gregorian::date名称空间(其中定义了path类)中，
// 也可以放在SF名称空间中。这里我们选择了SF名称空间。
namespace SF {
    void serialize(SF::Archive & ar, boost::gregorian::date & dt)
    {
        if (ar.isWrite())
        {
            // Code for serializing.
            std::ostringstream os;
            os << dt;
            std::string s = os.str();
            ar & s;
        }
        else
        {
            // Code for deserializing.
            std::string s;
            ar & s;
            std::istringstream is(s);
            is >> dt;
        }
    }
}
```
二进制数据
要发送二进制数据块，可以使用RCF::ByteBuffer类：
```cpp
RCF_BEGIN(I_Echo, "I_Echo")
    RCF_METHOD_R1(RCF::ByteBuffer, Echo, RCF::ByteBuffer)
RCF_END(I_Echo)
class EchoImpl
{
public:
    RCF::ByteBuffer Echo(RCF::ByteBuffer byteBuffer)
    {
        return byteBuffer;
    }
};
} 
int main()
{
    EchoImpl echoImpl;
    RCF::RcfServer server( RCF::TcpEndpoint(0));
    server.bind<I_Echo>(echoImpl);
    server.start();
    int port = server.getIpServerTransport().getPort();
    // Create and fill a 500 kb byte buffer.
    RCF::ByteBuffer byteBuffer(500*1024);
    for (std::size_t i=0; i<byteBuffer.getLength(); ++i)
    {
        byteBuffer.getPtr()[i] = char(i % 256);
    }
    RcfClient<I_Echo> client(( RCF::TcpEndpoint(port) ));
    // Echo it.
    RCF::ByteBuffer byteBuffer2 = client.Echo(byteBuffer);
    return 0;
}
```
std::string或std::vector<char>可以用于相同的目的。</char>但是，RCF::ByteBuffer的序列化和封送处理效率要高得多(参见性能)。使用RCF::ByteBuffer，在连接的两端都不会对数据进行任何复制。

可移植性
因为c++没有规定其基本类型的大小，所以当服务器和客户机部署在不同的平台上时，可能会出现序列化错误。例如，考虑以下接口：
```cpp
RCF_BEGIN(I_Echo, "I_Echo")
    RCF_METHOD_R1(std::size_t, Echo, std::size_t)
RCF_END(I_Echo)
```
Echo()方法使用std::size_t作为参数和返回值。

不幸的是，std::size_t在不同的平台上有不同的含义。例如，32位Visual c++编译器认为std::size_t是32位类型，而64位Visual c++编译器认为std::size_t是64位类型。因此，如果使用带有std::size_t参数的方法从32位客户机对64位服务器进行远程调用，将引发运行时序列化错误。

当在32位和64位版本的gcc中使用long类型时，也会出现同样的问题。32位gcc认为long是32位类型，而64位gcc认为long是64位类型。

在这些情况下，正确的方法是对保证位大小的算术类型使用typedefs。特别是，标准c++ <cstdint>头提供了许多有用的类型defs，包括以下内容。</cstdint>
类型   描述
std::int16_t	16 bit signed integer
std::uint16_t	16 bit unsigned integer
std::int32_t	32 bit signed integer
std::uint32_t	32 bit unsigned integer
std::int64_t	64 bit signed integer
std::uint64_t	64 bit unsigned integer
为了确保可移植性，上面的std::size_t示例应该重写为：
```cpp
RCF_BEGIN(I_Echo, "I_Echo")
    RCF_METHOD_R1(std::uint32_t, Echo, std::uint32_t)
    RCF_METHOD_R1(std::uint64_t, Echo, std::uint64_t)
RCF_END(I_Echo)
```
从磁盘到磁盘的序列化
RCF序列化可用于在磁盘和磁盘之间序列化对象。为此，使用SF::OBinaryStream和SF::IBinaryStream：
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