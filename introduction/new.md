<!--
 * @Author: haoluo
 * @Date: 2019-07-12 14:10:54
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-15 11:26:07
 * @Description: file content
 -->
## 3.0版本有什么新功能？
`RCF 3.0` 已经进行了现代化，利用了 C++11、C++14 和 C++17 更新中 C++ 标准中大量的语言和库特性。

此版本 RCF 不再依赖 [Boost](http://www.boost.org/) 库 。Boost 类和函数的所有用法，如 `boost::shared_ptr<>` 和 `boost::bind()`，都已被标准 C++ 对应项替换。
<!--修改 Building RCF 链接-->
其结果是，此版本RCF 将不再构建在旧的 C++ 编译器上。[Building RCF]() 这一章包含了一个受支持编译器的列表。无法升级到受支持编译器的用户将需要继续使用 RCF 2.2。

对于3.0版本，我们还全面检查了所有文档，并为 RCF public 类添加了参考文档。
<!--修改 发布说明 链接-->
有关更改的详细列表，请参阅[发布说明]()。