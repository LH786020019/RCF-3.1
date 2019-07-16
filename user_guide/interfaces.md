<!--
 * @Author: haoluo
 * @Date: 2019-07-15 11:42:01
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-16 12:38:27
 * @Description: file content
 -->
## 接口
`RCF` 中的远程调用基于接口的概念。RCF 接口使用 `RCF_BEGIN()`、`RCF_METHOD_<xx>()` 和 `RCF_END()` 宏来指定。一个接口指定一些远程方法，以及这些方法的参数和返回值。

下面是一个简单的 RCF 接口的例子：
```cpp
RCF_BEGIN(I_Calculator)
    RCF_METHOD_R2(int, Add, int, int)
    RCF_METHOD_V3(void, Multiply, int, int, int&)
RCF_END(I_Calculator)
```
由于 C++ preprocessor(预处理器) 的限制，`RCF_METHOD_<xx>()` 宏的名称取决于该方法的参数数量，以及该方法是否具有返回类型。命名约定如下：

- `RCF_METHOD_{V|R}{<n>}()` —— 定义一个 RCF 方法，它返回 `void(V)` 或 `non-void(R)`，并接受 `n` 个参数，其中 `n` 的取值范围为 0 到 15。

例如，`RCF_METHOD_V0()` 定义了一个带有 void 返回类型且没有参数的方法，而 `RCF_METHOD_R5()` 定义了一个带有非 void 返回类型且带有 5 个参数的方法。

注意，参数名可能不会出现在接口定义中。你可以把它们添加为注释：
```cpp
RCF_BEGIN(I_Calculator)
    RCF_METHOD_R2(
        int,                // Returns sum of two numbers.
            Add, 
                int,        // First number.
                int)        // Second number.
    RCF_METHOD_V3(
        void, 
            Multiply, 
                int,        // First number.
                int,        // Second number.
                int&)       // Returns product of two numbers.
RCF_END(I_Calculator)
```
一个方法的返回和参数类型可以是任意 C++ 数据类型。但是，对于使用的任何类型，都必须有一个序列化函数可用（ 请参阅[序列化](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/serialization.md) ）。RCF 为大多数标准库数据类型提供了内置的序列化函数，但是对于应用程序特定的数据类型，您需要编写自己的序列化函数。

一个 RCF 方法的参数可以是值、const 引用或非 const 引用。要从一个远程调用返回数据，您可以使用返回类型，也可以使用非 const 引用。上面的示例演示了如何使用 `Add()` 的返回类型和 `Multiply()` 的非 const 引用参数。

可以使用指针作为一个远程调用的参数。但是，为了避免手工内存管理，强烈建议您使用 C++ 智能指针类，比如 `std::unique_ptr<>` 和 `std::shared_ptr<>`。

一个 RCF 接口中的方法是根据它们出现的顺序来标识的。每个方法都有一个方法 ID 号，第一个方法从 0 开始，然后每个方法递增 1。由于方法 ID 号标识方法，从版本控制的角度来看（ 请参阅[版本控制](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/versioning.md) ），重要的是不要在一个 RCF 接口中重新排列方法的顺序，或者在现有方法之间插入方法（ 如果已经使用该接口部署了 client 和 server ）。

在一个 RCF 接口中可以出现的方法的数量是有限制的。这个限制默认设置为100，可以通过在您的 build 中定义 `RCF_MAX_METHOD_COUNT` 来调整（ 请参阅[构建 RCF](https://love2.io/@lh786020019/doc/RCF-3.1/building_RCF/index.md) ）。