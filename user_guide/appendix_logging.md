<!--
 * @Author: haoluo
 * @Date: 2019-07-16 10:35:08
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-16 10:37:13
 * @Description: file content
 -->
##　附录 —— 日志
RCF有一个可配置的日志子系统，可以通过RCF::enableLogging()和RCF::disableLogging()函数控制该子系统。要使用这些函数，需要包含<rcf log。<="" span="">hpp >头。</rcf>

默认情况下，日志是禁用的。要启用日志记录，请调用RCF::enableLogging()。enableLogging()接受两个可选参数，允许您指定日志级别和日志目标。
```cpp
    // Using default values for log target and log level.
    RCF::enableLogging();
    // Using custom values for log target and log level.
    int logLevel = 2;
    RCF::enableLogging(RCF::LogToDebugWindow(), logLevel);
```
日志级别可以从0(完全没有日志记录)到4(详细日志记录)。默认日志级别为2。

日志目标参数可以是以下参数之一：
日志目标    日志输出位置
RCF::LogToDebugWindow()	仅Windows。日志输出出现在Visual Studio调试输出窗口中。
RCF::LogToStdout()	日志输出出现在标准输出上。
RCF::LogToFile(const std::string & logFilePath)	日志输出出现在指定的文件中。
RCF::LogToFunc(std::function<void(const RCF::ByteBuffer &)>)	日志输出传递给用户定义的函数。
在Windows平台上，默认的日志目标是RCF::LogToDebugWindow。

在非windows平台上，默认的日志目标是RCF::LogToStdout。

最后，要禁用日志记录，调用RCF::disableLogging()：
```cpp
    // Disable logging.
    RCF::disableLogging();
```
RCF::enableLogging()和RCF::disableLogging()在内部是线程安全的，可以由多个线程并发调用。