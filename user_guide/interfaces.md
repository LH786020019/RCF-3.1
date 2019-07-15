<!--
 * @Author: haoluo
 * @Date: 2019-07-15 11:42:01
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-15 11:42:01
 * @Description: file content
 -->
## 接口
RCF中的远程调用基于接口的概念。RCF接口使用RCF_BEGIN()、RCF_METHOD_<xx>()和RCF_END()宏指定。</xx>接口指定许多远程方法，以及这些方法的参数和返回值。

下面是一个简单的RCF接口的例子：
```cpp
RCF_BEGIN(I_Calculator)
    RCF_METHOD_R2(int, Add, int, int)
    RCF_METHOD_V3(void, Multiply, int, int, int&)
RCF_END(I_Calculator)
```
由于c++预处理器的限制，RCF_METHOD_<xx>()宏的名称取决于该方法的参数数量，以及该方法是否具有返回类型。</xx>命名约定如下:

RCF_METHOD_{V|R}{<n>}() -定义一个RCF方法返回void (V)或non-void (R)，并接受n个参数，其中n的取值范围为0到15。</n>

例如，RCF_METHOD_V0()定义了一个带有void返回类型且没有参数的方法，而RCF_METHOD_R5()定义了一个带有非void返回类型且带有5个参数的方法。

注意，参数名可能不会出现在接口定义中。你可以把它们添加为评论：
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
方法的返回和参数类型可以是任意c++数据类型。但是，对于使用的任何类型，都必须有一个序列化函数可用(请参阅序列化)。RCF为大多数标准库数据类型提供内置的序列化函数，但是对于特定于应用程序的数据类型，您需要编写自己的序列化函数。

RCF方法的参数可以是值、const引用或非const引用。要从远程调用返回数据，可以使用返回类型，也可以使用非const引用。上面的示例演示了如何使用Add()的返回类型和Multiply()的非const引用参数。

可以使用指针作为远程调用的参数。但是，为了避免手工内存管理，强烈建议您使用c++智能指针类，比如std::unique_ptr<>和std::shared_ptr<>。

RCF接口中的方法是根据它们出现的顺序来标识的。每个方法都有一个方法ID号，第一个方法从0开始，然后每个方法递增1。由于方法ID号标识方法，从版本控制的角度来看(请参阅版本控制)，重要的是不要在RCF接口中重新排列方法的顺序，或者在现有方法之间插入方法(如果已经使用该接口部署了客户机和服务器)。

在一个RCF接口中可以出现的方法的数量是有限制的。这个限制默认设置为100，可以通过在构建中定义RCF_MAX_METHOD_COUNT来调整(请参见构建RCF)。