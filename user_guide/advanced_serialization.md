<!--
 * @Author: haoluo
 * @Date: 2019-07-16 10:27:44
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-17 18:41:39
 * @Description: file content
 -->
## 高级序列化
序列化泛型 C++ 对象本身是一项复杂的任务。本节描述 RCF 的内部序列化框架的一些更高级的特性。

### 1. 多态序列化(Polymorphic Serialization)
RCF 将自动检测和序列化多态指针和引用，作为完全派生的类型。然而，要做到这一点，RCF 需要配置两条关于正在序列化的多态类型的信息。首先，RCF 需要派生类型的运行时标识符字符串。其次，它需要知道派生类型将通过哪些基类型进行序列化。

下面是一个多态类层次结构的例子，带有相关的序列化函数：
```cpp
class A{
public:
    virtual ~A(){}
    void serialize(SF::Archive &ar){
        ar & mA;
    }
    int mA = 0;
};
class B : public A{
public:
    void serialize(SF::Archive &ar){
        SF::serializeParent<A>(ar, *this);
        ar & mB;
    }
    int mB = 0;
};
class C : public B{
public:
    void serialize(SF::Archive &ar){
        SF::serializeParent<B>(ar, *this);
        ar & mC;
    }
    int mC = 0;
};
```
注意，[SF::serializeParent()](http://www.deltavsoft.com/doc/_serialize_parent_8hpp.html#aa7d34330e49d5fcff85faca344c05c41) 用于调用基类序列化代码。如果您尝试直接序列化父类，例如通过调用 `ar & static_cast<A&>(*this)`， RCF 将检测父类实际上是一个派生类，并再次尝试序列化派生类。

现在考虑下面的 RCF 接口，它使用了一个包含多态 `A` 指针的类 `X`：
```cpp
class X{
public:
    void serialize(SF::Archive &ar){
        ar & mAPtr;
    }
    std::shared_ptr<A> mAPtr;
};
RCF_BEGIN(I_Echo, "I_Echo")
    RCF_METHOD_R1(X, Echo, X)
RCF_END(I_Echo)
```
为了多态序列化 `X::mAPtr`，我们需要做两件事：
- 使用 `SF::registerType()` 为 `B` 和 `C` 类设置运行时标识符。当序列化 `B` 和 `C` 对象时，运行时标识符将包含在序列化的归档文件中，并允许反序列化代码构造适当类型的对象。
- 使用 `SF::registerBaseAndDerived()` 指定派生类将通过哪些基类进行序列化。

在我们的例子中，以下代码就足够了：
```cpp
    // 注册多态类型。
    SF::registerType<B>("B");
    SF::registerType<C>("C");
    // 多态类型的寄存器基/派生关系。
    SF::registerBaseAndDerived<A, B>();
    SF::registerBaseAndDerived<A, C>();
```
此代码需要在 client 和 server 启动时执行。一旦执行，我们可以对 `Echo()` 方法进行远程调用，多态 `A` 指针将被序列化和反序列化，作为完全派生的类型。
```cpp
        RcfClient<I_Echo> client(( RCF::TcpEndpoint(port)));
        X x1;
        x1.mAPtr.reset( new B() );
        X x2 = client.Echo(x1);
        x1.mAPtr.reset( new C() );
        x2 = client.Echo(x1);
```

### 2. 指针跟踪(Pointer Tracking)
如果将指向同一对象的指针序列化两次，RCF 默认情况下将序列化整个对象两次。这意味着当指针反序列化时，它们将指向两个不同的对象。在大多数应用程序中，这通常不是问题。然而，一些应用程序可能希望反序列化代码创建两个指向相同对象的指针。

RCF 通过一个指针跟踪概念来支持这一点，在这个概念中，无论序列化了多少指向同一个对象的指针，此对象都只序列化一次。在反序列化时，只创建一个对象，然后可以反序列化多个指针，指向同一个对象。

为了演示指针跟踪，这里有一个带有 `Echo()` 函数的 `I_Echo` 接口，它接受一对 `std::shared_ptr<>` 对象：
```cpp
    typedef 
        std::pair< std::shared_ptr<std::string>, std::shared_ptr<std::string> >
        PointerPair;
    RCF_BEGIN(I_Echo, "I_Echo")
        RCF_METHOD_R1(PointerPair, Echo, PointerPair)
    RCF_END(I_Echo2)
```
下面是调用 `Echo()` 的 client 代码：
```cpp
        std::shared_ptr<std::string> ptr1( new std::string("one"));
        std::shared_ptr<std::string> ptr2( ptr1 );
        PointerPair ret = client.Echo( std::make_pair(ptr1, ptr2));
```
如果我们使用一对指向同一 `std::string` 的 `shared_ptr<>` 对象调用 `Echo()`，我们会发现返回的 `pair` 指向两个不同的 `std::string` 对象。为了让它们指向相同的 `std::string`，我们可以在 client 启用指针跟踪：
```cpp
        RcfClient<I_Echo> client(( RCF::TcpEndpoint(port)));
        client.getClientStub().setEnableSfPointerTracking(true);
```
，以及在 server 端启用：
```cpp
class EchoImpl{
public:
    template<typename T>
    T Echo(T t){
        RCF::getCurrentRcfSession().setEnableSfPointerTracking(true);
        return t;
    }
};
```
两个返回的 `shared_ptr<>` 对象现在将指向同一个实例。

指针跟踪是一个相对昂贵的特性，只有在应用程序需要时才应该启用它。指针跟踪要求序列化框架跟踪正在与一个归档进行序列化的所有指针和值，这会导致显著的性能开销。

### 3. 可互换的类型(Interchangeable Types)
序列化和反序列化倾向于对称地执行，因为序列化代码将类型 `T` 写入一个归档，而反序列化代码从此归档中读取同一类型 `T`。

然而，对称不是序列化的必要条件。如果两个类 `A` 和 `B` 具有序列化函数，并且它们序列化相同数量和类型的成员，则可以将一个 `A` 实例序列化为一个归档，并从此归档反序列化为一个 `B` 实例。本质上，从序列化的角度来看，类 A 和 B 是可互换的。

RCF 保证了许多类型的序列化可互换性。例如，对于任何类型的 `T`, `T` 和 `T *` 都是可互换的。所以你可以序列化一个指针，然后反序列化为一个值：
```cpp

    // 序列化一个指针
    int * pn = new int(5);
    std::ostringstream ostr;
    SF::OBinaryStream(ostr) << pn;
    delete pn;
    // 反序列化为一个值
    int m = 0;
    std::istringstream istr(ostr.str());
    SF::IBinaryStream(istr) >> m;
    // m is now 5
```
此外，智能指针(smart pointer)可以与本机指针(native pointer)互换，因此您可以序列化一个 `std::shared_ptr<T>`，然后将其反序列化为一个值 `T`。下面是另一个例子：
```cpp

// X 的 Client 端实现，使用 int
class X{
public:
    int n;
    void serialize(SF::Archive & ar, unsigned int){
        ar & n;
    }
};
```
```cpp
// X 的 Server-side 端实现，使用 shared_ptr<int>
class X{
public:
    std::shared_ptr<int> spn;
    void serialize(SF::Archive & ar){
        ar & spn;
    }
};
```
```cpp
// 即使使用不同的 X 实现，Client 和 Server 仍然可以通过这个接口进行交互。
RCF_BEGIN(I_EchoX,"I_EchoX")
    RCF_METHOD_R1(X, Echo, X)
RCF_END(I_EchoX)
```
这样做的一个重要结果是，如果您在 RCF 接口中开始使用 `T`、`T*`、`std::shared_ptr<T>`、`std::unique_ptr<T>` 或其他一些智能指针，那么总是可以在稍后的某个时间点更改它，而不会破坏向后兼容性。

以下是 RCF 确保可互换的类型的类：
- `T`, `T *`, `std::unique_ptr<T>`, `std::shared_ptr<T>`, `boost::scoped_ptr<T>`, `boost::shared_ptr<T>`
- 等价大小的整数类型，例如 `unsigned int` 和 `int`
- `C++98 enum` 和 32 位整数类型
- `T` 的 STL 容器，其中 `T` 是 non-primitive
- `T` 的 STL 容器(`std::vector<>`和`std::basic_string<>`除外)，其中 `T` 是 primitive
- `std::vector<T>` 和 `std::basic_string<T>`，其中 `T` 是 primitive
- `std::string`, `std::vector<char>`, 和 `RCF::ByteBuffer`

### 4. Unicode 字符串
`std::wstring` 对象的序列化带来了一些可移植性问题，因为不同的平台对 `wchar_t` 有不同的定义，对于 `std::wstring` 有不同的 Unicode 编码。

RCF 在序列化之前将 `std::wstring` 对象转换为 `UTF-8` 来解决这个问题。
- 在具有 16 位 `wchar_t` 的平台上，RCF 假设传递给它的任何 `std::wstring` 都是用 `UTF-16` 编码的，并在 `UTF-16` 和 `UTF-8` 之间进行转换。
- 在具有 32 位 `wchar_t` 的平台上，RCF 假设传递给它的任何 `std::wstring` 都是用 `UTF-32` 编码的，并在 `UTF-32` 和 `UTF-8` 之间进行转换。

如果您的应用程序正在发送或接收 `std::wstring` 对象，这些对象不是用假设的 `UTF-16` 或 `UTF-32` 编码的，那么您应该知道跨平台的可移植性可能会受到影响。