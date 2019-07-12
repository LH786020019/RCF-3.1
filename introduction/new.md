<!--
 * @Author: haoluo
 * @Date: 2019-07-12 14:10:54
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-12 14:17:58
 * @Description: file content
 -->
### 3.0版本有什么新功能？
RCF 3.0已经进行了现代化，利用了c++标准、c++ 11、c++ 14和c++ 17更新中大量的语言和库特性。

RCF不再依赖Boost库。Boost类和函数的所有用法，如Boost::shared_ptr<>和Boost::bind()，都已被标准c++对应项替换。

其结果是，RCF将不再构建在旧的c++编译器上。关于构建RCF的部分包含了一个受支持编译器的列表。无法升级到受支持编译器的用户将需要继续使用RCF 2.2。

对于3.0版本，我们还全面检查了所有文档，并为公共RCF类添加了参考文档。

有关更改的详细列表，请参阅发布说明。