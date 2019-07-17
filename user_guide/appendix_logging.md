<!--
 * @Author: haoluo
 * @Date: 2019-07-16 10:35:08
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-17 17:05:10
 * @Description: file content
 -->
##　附录 —— 日志
RCF有一个可配置的日志子系统，可以通过 [RCF::enableLogging()](http://www.deltavsoft.com/doc/group___functions.html#gae3c2b491aff36faa0807e6ceec4d91dd) 和 [RCF::disableLogging()](http://www.deltavsoft.com/doc/group___functions.html#ga4bf396da0a9711d2c7c2c156e57c3a3c) 函数控制该子系统。要使用这些函数，您需要 include `<RCF/Log.hpp><RCF/Log.hpp>` 头文件。

默认情况下，日志是禁用的。要启用日志，请调用 [RCF::enableLogging()](http://www.deltavsoft.com/doc/group___functions.html#gae3c2b491aff36faa0807e6ceec4d91dd)。[RCF::enableLogging()](http://www.deltavsoft.com/doc/group___functions.html#gae3c2b491aff36faa0807e6ceec4d91dd) 接受两个可选参数，允许您指定日志级别和日志目标。
```cpp
    // 为日志目标和日志级别使用默认值。
    RCF::enableLogging();
    // 为日志目标和日志级别使用自定义值。
    int logLevel = 2;
    RCF::enableLogging(RCF::LogToDebugWindow(), logLevel);
```
日志级别可以从 `0`(完全没有日志记录) 到 `4`(详细日志记录)。默认日志级别为 `2`。

日志目标参数可以是以下参数之一：
|日志目标 |   日志输出位置|
|--|--|
|RCF::LogToDebugWindow()|	日志输出在 Visual Studio 调试输出窗口中。仅 Windows。|
|[RCF::LogToStdout()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_log_to_stdout.html)|	日志输出在标准输出上。|
|[RCF::LogToFile(const std::string & logFilePath)](http://www.deltavsoft.com/doc/class_r_c_f_1_1_log_to_file.html)|	日志输出在指定的文件中。|
|[RCF::LogToFunc(std::function<void(const RCF::ByteBuffer &)>)](http://www.deltavsoft.com/doc/class_r_c_f_1_1_log_to_func.html)|	日志输出传递给用户定义的函数。|
在 Windows 平台上，默认的日志目标是 `RCF::LogToDebugWindow`。

在非 Windows 平台上，默认的日志目标是 [RCF::LogToStdout()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_log_to_stdout.html)。

最后，要禁用日志，调用 [RCF::disableLogging()](http://www.deltavsoft.com/doc/group___functions.html#ga4bf396da0a9711d2c7c2c156e57c3a3c)：
```cpp
    // Disable logging.
    RCF::disableLogging();
```
[RCF::enableLogging()](http://www.deltavsoft.com/doc/group___functions.html#gae3c2b491aff36faa0807e6ceec4d91dd) 和 [RCF::disableLogging()](http://www.deltavsoft.com/doc/group___functions.html#ga4bf396da0a9711d2c7c2c156e57c3a3c) 在内部是线程安全的，可以由多个线程并发调用。