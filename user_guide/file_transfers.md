<!--
 * @Author: haoluo
 * @Date: 2019-07-16 10:16:36
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-17 15:04:31
 * @Description: file content
 -->
## 文件传输
RCF 内置了对传输文件的支持。虽然较小的文件可以在单个远程调用中轻松传输，例如使用 [RCF::ByteBuffer](http://www.deltavsoft.com/doc/class_r_c_f_1_1_byte_buffer.html) 参数，但是一旦文件大小超过连接的最大消息大小，这种技术就会失败。

要可靠地传输任意大小的文件，需要将文件分割成块，并在单独的调用中传输，接收方重新组装块以形成目标文件。此外，为了获得用于大传输的最大吞吐量速度，还需要同时运行网络 I/O 和磁盘 I/O。

当您使用 RCF 内置的文件传输功能时，这些细节会自动处理。

文件传输与远程调用发生在相同的连接上。因此，如果为 RCF client 启用加密或压缩（ 请参阅[传输协议](https://love2.io/@lh786020019/doc/RCF-3.1/user_guide/transports_protocols.md) ），这些设置也将应用于该 RCF client 执行的任何文件传输。

要使用文件传输功能，必须在您的 `build` 中定义 `RCF_FEATURE_FILETRANSFER=1`（ 参见[构建 RCF](https://love2.io/@lh786020019/doc/RCF-3.1/building_RCF/index.md) ）。

### 1. 文件下载
要从一个 `RcfServer` 下载一个文件，请使用 [RCF::ClientStub::downloadFile()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a651399c13f012a6c23c498ec17422930)。

[RCF::ClientStub::downloadFile()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a651399c13f012a6c23c498ec17422930) 的第一个参数是一个`下载 ID`，必须由 `RcfServer` 分配。通常， client 将对 server 进行远程调用，作为对远程调用的响应的一部分，`下载 ID` 由 server 端应用程序代码分配，方法是调用[RCF::RcfSession::configureDownload()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_session.html#a924041739db1137b9b0395f573a2ae44)。`下载 ID` 返回给 client。

然后 client 使用下载 ID 来实际下载该文件。
```cpp

// Server-side code.
RCF_BEGIN(I_PrintService, "")
    RCF_METHOD_V1(void, Print, const std::string&)
    RCF_METHOD_R0(std::string, GetDocument)
RCF_END(I_PrintService)
class PrintService{
public:
    void Print(const std::string& msg){
        std::cout << msg;
    }
    std::string GetDocument(){
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
[RCF::ClientStub::downloadFile()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a651399c13f012a6c23c498ec17422930) 还接受一个第三个可选的参数，类型为 [RCF::FileTransferOptions](http://www.deltavsoft.com/doc/class_r_c_f_1_1_file_transfer_options.html)。[RCF::FileTransferOptions](http://www.deltavsoft.com/doc/class_r_c_f_1_1_file_transfer_options.html) 包含自定义文件传输的更多选项，包括下载一个文件片段的能力，并为传输分配带宽控制。
```cpp
    RCF::FileTransferOptions transferOptions;
    transferOptions.mBandwidthLimitBps = 1024 * 1024; // 1 MB/sec
    client.getClientStub().downloadFile(downloadId, fileToDownloadTo, &transferOptions);
```
### 2. 文件上传
要将一个文件上传到一个 `RcfServer`，请使用 [RCF::ClientStub::uploadFile()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a8d2a5dbe6eabef001155c37037ab6984)。

上传文件的操作与下载类似。[RCF::ClientStub::uploadFile()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a8d2a5dbe6eabef001155c37037ab6984) 的第一个参数是由 client 分配的一个`上传 ID`，client 在随后的远程调用中使用它来标识上传。

一旦 [RCF::ClientStub::uploadFile()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a8d2a5dbe6eabef001155c37037ab6984) 返回，就可以在远程调用中传递`上传 ID` 作为参数，然后 server 端应用程序代码就可以使用`上传 ID` 检索上传的文件。
```cpp
// Server-side code.
RCF_BEGIN(I_PrintService, "")
    RCF_METHOD_V1(void, Print, const std::string&)
    RCF_METHOD_V1(void, AddDocument, const std::string&)
RCF_END(I_PrintService)
class PrintService{
public:
    void Print(const std::string& msg){
        std::cout << msg;
    }
    void AddDocument(const std::string& uploadId){
        RCF::Path pathToUpload = RCF::getCurrentRcfSession().getUploadPath(uploadId);
        // 应用程序特定的代码，用来处理上传的文件。
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
就像下载一样，[RCF::ClientStub::uploadFile()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a8d2a5dbe6eabef001155c37037ab6984) 接受一个第三个可选的参数，类型为 [RCF::FileTransferOptions](http://www.deltavsoft.com/doc/class_r_c_f_1_1_file_transfer_options.html)，带有自定义上传的更多选项。
```cpp
    RCF::FileTransferOptions transferOptions;
    transferOptions.mBandwidthLimitBps = 1024 * 1024; // 1 MB/sec
    client.getClientStub().uploadFile(uploadId, fileToUpload, &transferOptions);
```
注意，在 server 端，必须调用 [RCF::RcfServer::setUploadDirectory()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#a2a99bbc9b76aede330a85b5ff640db73) 来指定上传文件的位置。在上传之前必须设置此属性。

一旦文件被上载到 server，server 端应用程序代码就有责任管理其生命周期。

### 3. 恢复文件传输
#### 3.1 恢复下载
如果在下载过程中出现网络断开或其他问题，[RCF::ClientStub::downloadFile()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a651399c13f012a6c23c498ec17422930) 将抛出异常。要继续下载，重新建立连接后，可以再次调用 `downloadFile()`，使用相同的参数。RCF 将在下载被中断的地方恢复下载。

#### 3.2 重新上传
如果在上传过程中发生网络断开或其他问题，[RCF::ClientStub::uploadFile()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a8d2a5dbe6eabef001155c37037ab6984) 将抛出异常。要恢复上传，在重新建立连接之后，可以再次调用 `uploadFile()`，使用相同的参数。RCF 将在上传被中断的地方恢复上传。

### 4. 监控文件传输
在 client 端，要监视一个文件传输，使用 [RCF::FileTransferOptions](http://www.deltavsoft.com/doc/class_r_c_f_1_1_file_transfer_options.html) 参数，并将 [RCF::FileTransferOptions::mProgressCallback](http://www.deltavsoft.com/doc/class_r_c_f_1_1_file_transfer_options.html#a6b7f8fd92eba852b76b192818425a92e) 设置为一个应用程序定义的回调函数。回调函数将被重复调用，直到文件传输完成。
```cpp
void clientTransferProgress(const RCF::FileTransferProgress& progress, RCF::RemoteCallAction& action){
    double percentComplete = (double)progress.mBytesTransferredSoFar / (double)progress.mBytesTotalToTransfer;
    std::cout << "Download progress: " << percentComplete << "%" << std::endl;
}
```
```cpp
    RCF::FileTransferOptions transferOptions;
    transferOptions.mProgressCallback = &clientTransferProgress;
    client.getClientStub().downloadFile(downloadId, fileToDownloadTo);
```
在 server 端，您可以使用 [RCF::RcfServer::setDownloadProgressCallback()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#a560a85611e6eba9d21934a29dc802c12) 和 [RCF::RcfServer::setUploadProgressCallback()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#acbe5df6252176feec8e2666ef38bf1df) 来注册一个回调函数，该函数将在每次下载或上传块时调用，用于该 server 上的任何下载或上传。

从回调函数中，您可以检查关联的 [RCF::RcfSession](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_session.html)、[RCF::FileDownloadInfo](http://www.deltavsoft.com/doc/class_r_c_f_1_1_file_download_info.html) 和 [RCF::FileUploadInfo](http://www.deltavsoft.com/doc/class_r_c_f_1_1_file_upload_info.html) 对象。若要取消一个文件传输，将从回调函数中抛出一个异常。

### 5. 带宽限制
RCF 文件传输将自动消耗网络允许的最大带宽。然而，在某些情况下，您可能希望以有限的带宽运行文件传输，以限制总网络带宽的使用。

#### 5.1 Server 端带宽限制
您可以使用 [RCF::RcfServer::setUploadBandwidthLimit()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#abd33667252be8b2d2d839ce4e0118928) 和 [RCF::RcfServer::setDownloadBandwidthLimit()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#adbd72dda3ffa83def219bb47dd3d3e50) 来设置与 `RcfServer` 之间的所有文件传输所消耗的最大总带宽。

下面是一个设置 1Mbps 上传限制和 5Mbps 下载限制的例子：
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
例如，如果三个 client 连接到这个 server 并开始上传文件，那么这三个 client 所消耗的上传带宽将不允许超过 1Mbps。

同样，如果三个 client 同时下载文件，则 client 所消耗的总下载带宽将不允许超过 5Mbps。

#### 5.2 自定义 Server 端带宽限制
RCF 还支持在更细粒度的级别上设置带宽限制，应用程序定义的 client 子集共享特定的带宽配额。

例如，在连接到具有不同带宽特性的多个网络的 server 上，您可能希望根据正在进行文件传输的网络来限制文件传输带宽。为此，可以使用 [RCF::RcfServer::setUploadBandwidthQuotaCallback()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#a4a5fc6d431197f3b1a5e591f214fd54e) 和 [RCF::RcfServer::setDownloadBandwidthQuotaCallback()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#a8562d11aef1f35026b843573b9f8be43) 来配置一个回调函数，以便在启动文件传输时调用 `RcfServer`。

从回调函数中，您可以检查连接的 [RCF::RcfSession](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_session.html)，并为文件传输分配一个相应的带宽配额。

例如，假设我们要实现以下带宽限制：
- 1Mbps 上传带宽，用于从 `192.168.*.*` 的 TCP 连接。
- 56Kbps 上传带宽，用于从 `15.146.*.*` 的 TCP 连接。
- 无限的上传带宽，用于所有其他连接。

要实现这一点，我们需要三个单独的 [RCF::BandwidthQuota](http://www.deltavsoft.com/doc/class_r_c_f_1_1_bandwidth_quota.html) 对象，以表示上面列出的三个带宽配额。然后，我们可以使用 [RCF::RcfServer::setUploadBandwidthQuotaCallback()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#a4a5fc6d431197f3b1a5e591f214fd54e) 根据 client 端的 IP 地址来分配带宽配额：
```cpp
// 1 Mbps quota bucket.
RCF::BandwidthQuotaPtr quota_1_Mbps( new RCF::BandwidthQuota(1*1000*1000/8) );
// 56 Kbps quota bucket.
RCF::BandwidthQuotaPtr quota_56_Kbps( new RCF::BandwidthQuota(56*1000/8) );
// Unlimited quota bucket.
RCF::BandwidthQuotaPtr quota_unlimited( new RCF::BandwidthQuota(0) );
RCF::BandwidthQuotaPtr uploadBandwidthQuotaCb(RCF::RcfSession & session){
    // 使用 client IP 地址确定从哪个配额分配。
    const RCF::RemoteAddress & clientAddr = session.getClientAddress();
    const RCF::IpAddress & clientIpAddr = dynamic_cast<const RCF::IpAddress &>(clientAddr);
    if ( clientIpAddr.matches( RCF::IpAddress("192.168.0.0", 16) ) ) {
        return quota_1_Mbps;
    } else if ( clientIpAddr.matches( RCF::IpAddress("15.146.0.0", 16) ) ) {
        return quota_56_Kbps;
    } else {
        return quota_unlimited;
    }
}
```
```cpp
    // 指定一个自定义文件上传带宽限制。
    server.setUploadBandwidthQuotaCallback(&uploadBandwidthQuotaCb);
```
#### 5.3 Client 端带宽限制
要从 client 端分配带宽限制，请使用 [RCF::FileTransferOptions::mBandwidthLimitBps](http://www.deltavsoft.com/doc/class_r_c_f_1_1_file_transfer_options.html#ab065964692da6577dd82b01949e37e9d) 和 [RCF::FileTransferOptions::mBandwidthQuotaPtr](http://www.deltavsoft.com/doc/class_r_c_f_1_1_file_transfer_options.html#a8ae457362ca43356b770fc17ea02a854)。

[RCF::FileTransferOptions::mBandwidthLimitBps](http://www.deltavsoft.com/doc/class_r_c_f_1_1_file_transfer_options.html#ab065964692da6577dd82b01949e37e9d) 允许您为单个 client 端连接配置带宽限制：
```cpp
    RCF::FileTransferOptions transferOptions;
    transferOptions.mBandwidthLimitBps = 1024 * 1024; // 1 MB/sec
    client.getClientStub().downloadFile(downloadId, fileToDownloadTo, &transferOptions);
```
[RCF::FileTransferOptions::mBandwidthQuotaPtr](http://www.deltavsoft.com/doc/class_r_c_f_1_1_file_transfer_options.html#a8ae457362ca43356b770fc17ea02a854) 允许您在多个 client 连接之间共享带宽配额。下面是 3 个 client 端同时上传文件的例子，client 端使用的总带宽不允许超过 1 Mb/sec：
```cpp
        RcfClient<I_PrintService> client1(RCF::TcpEndpoint(50001));
        RcfClient<I_PrintService> client2(RCF::TcpEndpoint(50001));
        RcfClient<I_PrintService> client3(RCF::TcpEndpoint(50001));
        RCF::BandwidthQuotaPtr clientQuotaPtr(new RCF::BandwidthQuota(1024*1024));
        auto doUpload = [=](RcfClient<I_PrintService>& client){
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
        for ( std::thread& thread : clientThreads ) {
            thread.join();
        }
```