<!--
 * @Author: haoluo
 * @Date: 2019-07-16 10:16:36
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-16 10:21:17
 * @Description: file content
 -->
## 文件传输
RCF内置了对传输文件的支持。虽然较小的文件可以在单个远程调用中轻松传输，例如使用RCF::ByteBuffer参数，但是一旦文件大小超过连接的最大消息大小，这种技术就会失败。

要可靠地传输任意大小的文件，需要将文件分割成块，并在单独的调用中传输，接收方重新组装块以形成目标文件。此外，为了获得大传输的最大吞吐量速度，还需要同时运行网络I/O和磁盘I/O。

当您使用RCF内置的文件传输功能时，这些细节会自动处理。

文件传输与远程调用发生在相同的连接上。因此，如果为RCF客户机启用加密或压缩(请参阅传输协议)，这些设置也将应用于该RCF客户机执行的任何文件传输。

要使用文件传输功能，必须在构建中定义RCF_FEATURE_FILETRANSFER=1(参见构建RCF)。
文件下载
要从RcfServer下载文件，请使用RCF::ClientStub::downloadFile()。

downloadFile()的第一个参数是一个下载ID，必须由RcfServer分配。通常，客户机将对服务器进行远程调用，作为对远程调用的响应的一部分，下载ID由服务器端应用程序代码分配，方法是调用RCF::RcfSession::configureDownload()。下载ID返回给客户机。

然后客户机使用下载ID实际下载文件。
```cpp

// Server-side code.
RCF_BEGIN(I_PrintService, "")
    RCF_METHOD_V1(void, Print, const std::string&)
    RCF_METHOD_R0(std::string, GetDocument)
RCF_END(I_PrintService)
class PrintService
{
public:
    void Print(const std::string& msg)
    {
        std::cout << msg;
    }
    std::string GetDocument()
    {
        RCF::Path fileToDownload = "C:\\Document1.pdf";
        std::string downloadId = RCF::getCurrentRcfSession().configureDownload(fileToDownload);
        return downloadId;
    }
};
```
```cpp
    // Server-side code.
    RCF::RcfServer server(RCF::TcpEndpoint(50001));
    PrintService printService;
    server.bind<I_PrintService>(printService);
    server.start();
```
```cpp
    // Client-side code.
    RcfClient<I_PrintService> client(RCF::TcpEndpoint(50001));
    std::string downloadId = client.GetDocument();
    RCF::Path fileToDownloadTo = "C:\\Document1.pdf";
    client.getClientStub().downloadFile(downloadId, fileToDownloadTo);
```
downloadFile()还接受第三个可选参数，类型为RCF::FileTransferOptions。FileTransferOptions包含定制文件传输的更多选项，包括下载文件片段的能力，并为传输分配带宽控制。
```cpp
    RCF::FileTransferOptions transferOptions;
    transferOptions.mBandwidthLimitBps = 1024 * 1024; // 1 MB/sec
    client.getClientStub().downloadFile(downloadId, fileToDownloadTo, &transferOptions);
```
文件上传
要将文件上载到RcfServer，请使用RCF::ClientStub::uploadFile()。

上传文件的操作与下载类似。uploadFile()的第一个参数是由客户机分配的上载ID，客户机在随后的远程调用中使用它来标识上载。

一旦返回RCF::ClientStub::uploadFile()，就可以在远程调用中传递上载ID作为参数，然后服务器端应用程序代码就可以使用上载ID检索上载文件。
```cpp
// Server-side code.
RCF_BEGIN(I_PrintService, "")
    RCF_METHOD_V1(void, Print, const std::string&)
    RCF_METHOD_V1(void, AddDocument, const std::string&)
RCF_END(I_PrintService)
class PrintService
{
public:
    void Print(const std::string& msg)
    {
        std::cout << msg;
    }
    void AddDocument(const std::string& uploadId)
    {
        RCF::Path pathToUpload = RCF::getCurrentRcfSession().getUploadPath(uploadId);
        // Application-specific code to process the uploaded file.
        // ...
        namespace fs = std::experimental::filesystem;
        fs::remove(pathToUpload);
    }
};
```
```cpp
    // Server-side code.
    RCF::RcfServer server(RCF::TcpEndpoint(50001));
    PrintService printService;
    server.bind<I_PrintService>(printService);
    
    server.setUploadDirectory("C:\\MyApp\\Uploads");
    
    server.start();
```
```cpp
    // Client-side code.
    RcfClient<I_PrintService> client(RCF::TcpEndpoint(50001));
    std::string uploadId = RCF::generateUuid();
    RCF::Path fileToUpload = "C:\\Document1.pdf";
    client.getClientStub().uploadFile(uploadId, fileToUpload);
    client.AddDocument(uploadId);
```
就像下载一样，RCF::ClientStub::uploadFile()接受第三个可选参数，类型为RCF::FileTransferOptions，带有定制上传的更多选项。
```cpp
    RCF::FileTransferOptions transferOptions;
    transferOptions.mBandwidthLimitBps = 1024 * 1024; // 1 MB/sec
    client.getClientStub().uploadFile(uploadId, fileToUpload, &transferOptions);
```
注意，在服务器端，必须调用RCF::RcfServer::setUploadDirectory()来指定上载文件的位置。在上传之前必须设置此属性。

一旦文件被上载到服务器，服务器端应用程序代码就有责任管理其生存期。

恢复文件传输
恢复下载
如果在下载过程中出现网络断开或其他问题，RCF::ClientStub::downloadFile()将抛出异常。要继续下载，重新建立连接后，可以再次调用downloadFile()，使用相同的参数。RCF将在中断下载时恢复下载。

重新上传
如果在上传过程中发生网络断开或其他问题，RCF::ClientStub::uploadFile()将抛出异常。要恢复上传，在重新建立连接之后，可以再次调用uploadFile()，使用相同的参数。RCF将在上传被中断时恢复上传。

监控文件传输
在客户端，要监视文件传输，使用RCF::FileTransferOptions参数，并将RCF::FileTransferOptions::mProgressCallback设置为应用程序定义的回调函数。回调函数将重复调用，直到文件传输完成。
```cpp
void clientTransferProgress(const RCF::FileTransferProgress& progress, RCF::RemoteCallAction& action)
{
    double percentComplete = (double)progress.mBytesTransferredSoFar / (double)progress.mBytesTotalToTransfer;
    std::cout << "Download progress: " << percentComplete << "%" << std::endl;
}
```
```cpp
    RCF::FileTransferOptions transferOptions;
    transferOptions.mProgressCallback = &clientTransferProgress;
    client.getClientStub().downloadFile(downloadId, fileToDownloadTo);
```
在服务器端，您可以使用RCF::RcfServer::setDownloadProgressCallback()和RCF::RcfServer::setUploadProgressCallback()来注册一个回调函数，该函数将在每次下载或上载块时调用，用于该服务器上的任何下载或上载。

从回调函数中，您可以检查关联的RCF::RcfSession、RCF::FileDownloadInfo和RCF::FileUploadInfo对象。若要取消文件传输，请从回调中抛出异常。

带宽限制
RCF文件传输将自动消耗网络允许的最大带宽。然而，在某些情况下，您可能希望以有限的带宽运行文件传输，以限制总网络带宽的使用。

服务器端带宽限制
您可以使用RCF::RcfServer::setUploadBandwidthLimit()和RCF::RcfServer::setDownloadBandwidthLimit()来设置与RcfServer之间的所有文件传输所消耗的最大总带宽。

下面是一个设置1mbps上传限制和5mbps下载限制的例子：
```cpp
    RCF::RcfServer server( RCF::TcpEndpoint(50001) );
    // 1 Mbps upload limit.
    std::uint32_t serverUploadLimitMbps = 1;
    // Convert to bytes/second.
    std::uint32_t serverUploadLimitBps = serverUploadLimitMbps*1000*1000/8;
    server.setUploadBandwidthLimit(serverUploadLimitBps);
    // 5 Mbps upload limit.
    std::uint32_t serverDownloadLimitMbps = 5;
    
    // Convert to bytes/second.
    std::uint32_t serverDownloadLimitBps = serverDownloadLimitMbps*1000*1000/8;
    server.setUploadBandwidthLimit(serverDownloadLimitBps);
```
例如，如果三个客户机连接到这个服务器并开始上传文件，那么这三个客户机所消耗的上传带宽将不允许超过1m / s。

同样，如果三个客户机同时下载文件，则客户机所消耗的总下载带宽将不允许超过5mbps。

自定义服务器端带宽限制
RCF还支持在更细粒度的级别上设置带宽限制，应用程序定义的客户机子集共享特定的带宽配额。

例如，在连接到具有不同带宽特性的多个网络的服务器上，您可能希望根据正在进行文件传输的网络来限制文件传输带宽。为此，可以使用RCF::RcfServer::setUploadBandwidthQuotaCallback()和RCF::RcfServer::setDownloadBandwidthQuotaCallback()来配置一个回调函数，以便在启动文件传输时调用RcfServer。

从回调函数中，您可以检查连接的RCF::RcfSession，并为文件传输分配相应的带宽配额。

例如，假设我们要实现以下带宽限制:

1 Mbps上传带宽为TCP连接从192.168.*.*。
56 Kbps上传带宽为TCP连接从15.146.*.*。
无限的上传带宽为所有其他连接。
要实现这一点，我们需要三个单独的RCF::BandwidthQuota对象，以表示上面列出的三个带宽配额。然后，我们可以使用RCF::RcfServer::setUploadBandwidthQuotaCallback()根据客户端的IP地址分配带宽配额：
```cpp
// 1 Mbps quota bucket.
RCF::BandwidthQuotaPtr quota_1_Mbps( new RCF::BandwidthQuota(1*1000*1000/8) );
// 56 Kbps quota bucket.
RCF::BandwidthQuotaPtr quota_56_Kbps( new RCF::BandwidthQuota(56*1000/8) );
// Unlimited quota bucket.
RCF::BandwidthQuotaPtr quota_unlimited( new RCF::BandwidthQuota(0) );
RCF::BandwidthQuotaPtr uploadBandwidthQuotaCb(RCF::RcfSession & session)
{
    // Use clients IP address to determine which quota to allocate from.
    const RCF::RemoteAddress & clientAddr = session.getClientAddress();
    const RCF::IpAddress & clientIpAddr = dynamic_cast<const RCF::IpAddress &>(clientAddr);
    if ( clientIpAddr.matches( RCF::IpAddress("192.168.0.0", 16) ) )
    {
        return quota_1_Mbps;
    }
    else if ( clientIpAddr.matches( RCF::IpAddress("15.146.0.0", 16) ) )
    {
        return quota_56_Kbps;
    }
    else
    {
        return quota_unlimited;
    }
}
```
```cpp
    // Assign a custom file upload bandwidth limit.
    server.setUploadBandwidthQuotaCallback(&uploadBandwidthQuotaCb);
```
客户端带宽限制
要从客户端分配带宽限制，请使用RCF::FileTransferOptions::mBandwidthLimitBps和RCF::FileTransferOptions::mBandwidthQuotaPtr。

mBandwidthLimitBps允许您为单个客户端连接配置带宽限制：
```cpp
    RCF::FileTransferOptions transferOptions;
    transferOptions.mBandwidthLimitBps = 1024 * 1024; // 1 MB/sec
    client.getClientStub().downloadFile(downloadId, fileToDownloadTo, &transferOptions);
```
mBandwidthQuotaPtr允许您在多个客户机连接之间共享带宽配额。下面是3个客户端同时上传文件的例子，客户端使用的总带宽不允许超过1mb /秒：
```cpp
        RcfClient<I_PrintService> client1(RCF::TcpEndpoint(50001));
        RcfClient<I_PrintService> client2(RCF::TcpEndpoint(50001));
        RcfClient<I_PrintService> client3(RCF::TcpEndpoint(50001));
        RCF::BandwidthQuotaPtr clientQuotaPtr(new RCF::BandwidthQuota(1024*1024));
        auto doUpload = [=](RcfClient<I_PrintService>& client)
        {
            std::string uploadId                    = RCF::generateUuid();
            RCF::Path fileToUpload                  = "C:\\Document1.pdf";
            RCF::FileTransferOptions transferOptions;
            transferOptions.mBandwidthQuotaPtr      = clientQuotaPtr;
            client.getClientStub().uploadFile(uploadId, fileToUpload, &transferOptions);
        };
        std::vector<std::thread> clientThreads;
        clientThreads.push_back(std::thread(std::bind(doUpload, std::ref(client1))));
        clientThreads.push_back(std::thread(std::bind(doUpload, std::ref(client2))));
        clientThreads.push_back(std::thread(std::bind(doUpload, std::ref(client3))));
        for ( std::thread& thread : clientThreads )
        {
            thread.join();
        }
```