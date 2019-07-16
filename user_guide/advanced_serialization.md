<!--
 * @Author: haoluo
 * @Date: 2019-07-16 10:27:44
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-16 10:31:16
 * @Description: file content
 -->
## 高级序列化
序列化泛型c++对象本身是一项复杂的任务。本节描述RCF的内部序列化框架的一些更高级的特性。

多态序列化
RCF将自动检测和序列化多态指针和引用，作为完全派生的类型。然而，要做到这一点，RCF需要配置两条关于正在序列化的多态类型的信息。首先，RCF需要派生类型的运行时标识符字符串。其次，它需要知道派生类型将通过哪些基类型进行序列化。

下面是一个多态类层次结构的例子，带有相关的序列化函数：
```cpp
class A
{
public:
    virtual ~A() 
    {}
    void serialize(SF::Archive &ar)
    {
        ar & mA;
    }
    int mA = 0;
};
class B : public A
{
public:
    void serialize(SF::Archive &ar)
    {
        SF::serializeParent<A>(ar, *this);
        ar & mB;
    }
    int mB = 0;
};
class C : public B
{
public:
    void serialize(SF::Archive &ar)
    {
        SF::serializeParent<B>(ar, *this);
        ar & mC;
    }
    int mC = 0;
};
```
注意，SF::serializeParent()用于调用基类序列化代码。如果您尝试直接序列化父类，例如通过调用ar & static_cast< a&> (*this)， RCF将检测父类实际上是一个派生类，并再次尝试序列化派生类。

现在考虑下面的RCF接口，它使用了一个包含多态a指针的类X：
```cpp
class X
{
public:
    
    void serialize(SF::Archive &ar)
    {
        ar & mAPtr;
    }
    std::shared_ptr<A> mAPtr;
};
RCF_BEGIN(I_Echo, "I_Echo")
    RCF_METHOD_R1(X, Echo, X)
RCF_END(I_Echo)
```
为了多态序列化X::mAPtr，我们需要做两件事:

使用SF::registerType()为B和C类设置运行时标识符。当序列化B和C对象时，运行时标识符将包含在序列化的归档文件中，并允许反序列化代码构造适当类型的对象。
使用SF::registerBaseAndDerived()指定派生类将通过哪些基类进行序列化。
在我们的例子中，以下代码就足够了：
```cpp
    // Register polymorphic types.
    SF::registerType<B>("B");
    SF::registerType<C>("C");
    // Register base/derived relationships for polymorphic types.
    SF::registerBaseAndDerived<A, B>();
    SF::registerBaseAndDerived<A, C>();
```
此代码需要在客户机和服务器启动时执行。一旦执行，我们可以对Echo()方法进行远程调用，多态A指针将被序列化和反序列化，成为完全派生的类型。
```cpp
        RcfClient<I_Echo> client(( RCF::TcpEndpoint(port)));
        X x1;
        x1.mAPtr.reset( new B() );
        X x2 = client.Echo(x1);
        x1.mAPtr.reset( new C() );
        x2 = client.Echo(x1);
```
指针跟踪
如果将指向同一对象的指针序列化两次，RCF默认情况下将序列化整个对象两次。这意味着当指针反序列化时，它们将指向两个不同的对象。在大多数应用程序中，这通常不是问题。然而，一些应用程序可能希望反序列化代码创建两个指向相同对象的指针。

RCF通过一个指针跟踪概念来支持这一点，在这个概念中，无论序列化了多少指向对象的指针，对象都只序列化一次。在反序列化时，只创建一个对象，然后可以反序列化多个指针，指向同一个对象。

为了演示指针跟踪，这里有一个带有Echo()函数的I_Echo接口，它接受一对std::shared_ptr<>对象：
```cpp
    typedef 
        std::pair< std::shared_ptr<std::string>, std::shared_ptr<std::string> >
        PointerPair;
    RCF_BEGIN(I_Echo, "I_Echo")
        RCF_METHOD_R1(PointerPair, Echo, PointerPair)
    RCF_END(I_Echo2)
```
下面是调用Echo()的客户端代码：
```cpp
        std::shared_ptr<std::string> ptr1( new std::string("one"));
        std::shared_ptr<std::string> ptr2( ptr1 );
        PointerPair ret = client.Echo( std::make_pair(ptr1, ptr2));
```
如果我们使用一对指向相同std::string的shared_ptr<>对象调用Echo()，我们会发现返回的对指向两个不同的std::string对象。为了让它们指向相同的std::string，我们可以在客户端启用指针跟踪：
```cpp
        RcfClient<I_Echo> client(( RCF::TcpEndpoint(port)));
        client.getClientStub().setEnableSfPointerTracking(true);
```
，以及服务器端：
```cpp
class EchoImpl
{
public:
    template<typename T>
    T Echo(T t)
    {
        RCF::getCurrentRcfSession().setEnableSfPointerTracking(true);
        return t;
    }
};
```
两个返回的shared_ptr<>对象现在将指向同一个实例。

指针跟踪是一个相对昂贵的特性，只有在应用程序需要时才应该启用它。指针跟踪要求序列化框架跟踪正在与存档进行序列化的所有指针和值，这会导致显著的性能开销。

可互换的类型
序列化和反序列化倾向于对称地执行，因为序列化代码将类型T写入存档，而反序列化代码从存档中读取相同类型T。

然而，对称不是序列化的必要条件。如果两个类A和B具有序列化功能，能够序列化相同数量和类型的成员，则可以将A实例序列化为存档，并从存档反序列化B实例。本质上，从序列化的角度来看，类A和B是可互换的。

RCF保证了许多类型的序列化可互换性。例如，对于任何类型的T, T和T *都是可互换的。所以你可以序列化一个指针，然后反序列化为一个值：
```cpp

    // Serializing a pointer.
    int * pn = new int(5);
    std::ostringstream ostr;
    SF::OBinaryStream(ostr) << pn;
    delete pn;
    // Deserializing a value.
    int m = 0;
    std::istringstream istr(ostr.str());
    SF::IBinaryStream(istr) >> m;
    // m is now 5
```
此外，智能指针可以与本机指针互换，因此您可以序列化std::shared_ptr<t>，然后将其反序列化为一个值T。</t>
```cpp

// Client-side implementation of X, using int.
class X
{
public:
    int n;
    void serialize(SF::Archive & ar, unsigned int)
    {
        ar & n;
    }
};
```
```cpp
// Server-side implementation of X, using shared_ptr<int>.
class X
{
public:
    std::shared_ptr<int> spn;
    void serialize(SF::Archive & ar)
    {
        ar & spn;
    }
};
```
```cpp
// Even with different X implementations, client and server can still 
// interact through this interface.
RCF_BEGIN(I_EchoX,"I_EchoX")
RCF_METHOD_R1(X, Echo, X)
RCF_END(I_EchoX)
```
这样做的一个重要结果是，如果您在RCF接口中开始使用T, T*， std::shared_ptr<t>， std::unique_ptr<t>或其他一些智能指针，那么总是可以在稍后的某个时间点更改它，而不会破坏向后兼容性。</t></t>

以下是RCF保证可互换的类型类:

T, T *， std::unique_ptr<t>， std::shared_ptr<t>， boost::scoped_ptr<t>， boost::shared_ptr<t></t></t></t></t>
等价大小的整数类型，例如无符号整型和整型
c++ 98枚举和32位整数类型
T的STL容器，其中T是非基元的
T的STL容器(std::vector<>和std::basic_string<>除外)，其中T是基元
std::vector<t> and std::basic_string<t>，其中T是基元</t></t>
std::string, std::vector<char>， RCF::ByteBuffer</char>
Unicode字符串
std::wstring对象的序列化带来了一些可移植性问题，因为不同的平台对wchar_t有不同的定义，std::wstring有不同的Unicode编码。

RCF在序列化之前将std::wstring对象转换为UTF-8来解决这个问题。

在具有16位wchar_t的平台上，RCF假设传递给它的任何std::wstring都是用UTF-16编码的，并在UTF-16和UTF-8之间进行转换。
在32位wchar_t平台上，RCF假设传递给它的任何std::wstring都是用UTF-32编码的，并在UTF-32和UTF-8之间进行转换。
如果您的应用程序正在发送或接收std::wstring对象，这些对象不是用假定的UTF-16或UTF-32编码的，那么您应该知道跨平台的可移植性可能会受到影响。