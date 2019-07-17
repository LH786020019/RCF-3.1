<!--
 * @Author: haoluo
 * @Date: 2019-07-16 09:18:41
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-17 18:45:08
 * @Description: file content
 -->
## 传输协议
传输协议用于转换通过传输的数据。RCF 使用传输协议为远程调用提供身份验证、加密和压缩。

RCF 目前支持 `NTLM`、`Kerberos`、`Negotiate` 和 `SSL` 传输协议。`NTLM`、`Kerberos` 和 `Negotiate` 只在 Windows 平台上受支持，而 `SSL` 在所有平台上都受支持。

此外，RCF 还支持基于 `Zlib` 的远程调用压缩。

传输协议是通过调用 [RCF::ClientStub::setTransportProtocol()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a3e1cbd34dc1006000319356737008a63) 在一个 client 连接上配置的：
```cpp
        RcfClient<I_PrintService> client(( RCF::TcpEndpoint(port) ));
        client.getClientStub().setTransportProtocol(RCF::Tp_Ntlm);
```
在一个 client 连接的 server 会话中，可以调用 [RCF::RcfSession::getTransportProtocol()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_session.html#a282b5988695fbe26d63ccc6ec4f97970) 来确定 client 正在使用的传输协议：
```cpp
        RCF::RcfSession & session = RCF::getCurrentRcfSession();
        RCF::TransportProtocol tp = session.getTransportProtocol();
```

### 1. NTLM
在一个 client 连接上配置 `NTLM`：
```cpp
    RcfClient<I_PrintService> client(( RCF::TcpEndpoint(port) ));
    client.getClientStub().setTransportProtocol(RCF::Tp_Ntlm);
    client.Print("Hello World");
```
在 server 端，您可以确定 client 的 Windows 用户名，并模拟(impersonate)它们：
```cpp
        std::string clientUsername = session.getClientUserName();
        RCF::SspiImpersonator impersonator(session);
        // 我们现在正在模拟 client，直到 `impersonator` 超出范围为止。
        // ...
```

### 2. Kerberos
在一个 client 连接上配置 `Kerberos`：
```cpp
        client.getClientStub().setTransportProtocol(RCF::Tp_Kerberos);
        client.getClientStub().setKerberosSpn("Domain\\ServerAccount");
        client.Print("Hello World");
```
注意，client 需要调用 `ClientStub::setKerberosSpn()` 来指定它期望 server 运行的用户名。这称为 server 的 `SPN` (Service Principal Name，服务主体名)，`Kerberos` 协议中使用它来实现相互身份验证。如果 server 不在此帐户下运行，连接将失败。

在 server 端，您可以确定 client 的 Windows 用户名，并模拟(impersonate)它们：
```cpp
        std::string clientUsername = session.getClientUserName();
        RCF::SspiImpersonator impersonator(session);
        // 我们现在正在模拟 client，直到 `impersonator` 超出范围为止。
        // ...
```

### 3. Negotiate
`Negotiate` 是 `NTLM` 和 `Kerberos` 协议之间的一个协商协议。如果可能，它将解析为 `Kerberos`，否则解析为 `NTLM`。

在一个 client 连接上配置 `Negotiate`：
```cpp
        client.getClientStub().setTransportProtocol(RCF::Tp_Negotiate);
        client.getClientStub().setKerberosSpn("Domain\\ServerAccount");
        client.Print("Hello World");
```
与 `Kerberos` 传输协议一样，您需要为 server 提供一个 `SPN`。

在 server 端，您可以确定 client 的 Windows 用户名，并模拟(impersonate)它们：
```cpp
        std::string clientUsername = session.getClientUserName();
        RCF::SspiImpersonator impersonator(session);
        // 我们现在正在模拟 client，直到 `impersonator` 超出范围为止。
        // ...
```

### 4. SSL
RCF 提供了两种 `SSL` 传输协议实现。一个基于跨平台 `OpenSSL` 库，另一个基于仅限 Windows 的 `Schannel` 包。

只有在定义了 `RCF_USE_OPENSSL` 之后，才可以使用 `OpenSSL` 支持。

`Schannel` 支持只在 Windows 构建中可用，不需要任何定义。

如果在 Windows 构建中定义 `RCF_USE_OPENSSL`, RCF将使用 `OpenSSL` 而不是 `Schannel`。如果你想使用 `Schannel`，尽管定义了 `RCF_USE_OPENSSL`，您可以使用 [RCF::RcfServer::setSslImplementation()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#af60d05d5e41fd7ed16ed3efe5f7c703b) 和 [RCF::ClientStub::setSslImplementation()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a01372c4b051eee29e6a4c6da97510b51) 函数，来为各个 server 和 client 设置 SSL 实现,或者使用 [RCF::Globals::setDefaultSslImplementation()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_globals.html#ac73afd1c6fe86c3982e8a9c4f62b4976) 函数，以设置 SSL 实现用于整个 RCF 运行时。

SSL 协议使用证书对 client 的 server 进行身份验证(也可以选择对 server 的 client 进行身份验证)。证书和证书验证功能由两种 SSL 传输协议实现以不同的方式处理，如下所述。

#### 4.1 Schannel
配置一个 server 以接受 SSL 连接，您需要提供一个 server 证书。RCF 提供了 [RCF::PfxCertificate](http://www.deltavsoft.com/doc/class_r_c_f_1_1_pfx_certificate.html) 类，用于从 `.pfx` 和 `.p12` 文件加载证书：
```cpp
        RCF::CertificatePtr serverCertPtr( new RCF::PfxCertificate(
            "C:\\serverCert.p12", 
            "password", 
            "CertificateName") );
        server.setCertificate(serverCertPtr);
```
RCF 还提供了 [RCF::StoreCertificate](http://www.deltavsoft.com/doc/class_r_c_f_1_1_store_certificate.html) 类，用于从 Windows 证书存储加载证书：
```cpp
        RCF::CertificatePtr serverCertPtr( new RCF::StoreCertificate(
            RCF::Cl_LocalMachine,
            RCF::Cs_My,
            "CertificateName") );
        server.setCertificate(serverCertPtr);
```
在 client 上，您需要提供验证 server 提供的证书的方法。有几种方法可以做到这一点。您可以让 `Schannel` 包应用它自己的内部验证逻辑，它将遵从本地安装的认证中心(certificate authorities)：
```cpp
        client.getClientStub().setTransportProtocol(RCF::Tp_Ssl);
        client.getClientStub().setEnableSchannelCertificateValidation("CertificateName");
        client.Print("Hello World");
```
您还可以自己提供一个特定的认证中心(certificate authorities)，用于验证 server 证书：
```cpp
        RCF::CertificatePtr caCertPtr( new RCF::PfxCertificate(
            "C:\\clientCaCertificate.p12", 
            "password", 
            "CaCertificatename"));
        client.getClientStub().setCaCertificate(caCertPtr);
```
您还可以编写自己的自定义证书验证逻辑：
```cpp
bool schannelValidateCert(RCF::Certificate * pCert){
    RCF::Win32Certificate * pWin32Cert = static_cast<RCF::Win32Certificate *>(pCert);
    if (pWin32Cert) {
        RCF::tstring certName = pWin32Cert->getCertificateName();
        RCF::tstring issuerName = pWin32Cert->getIssuerName();
        PCCERT_CONTEXT pContext = pWin32Cert->getWin32Context();
        // 用于检查和验证证书的自定义代码
        // ...
    }
    // 如果认为证书有效，则返回 true。否则，返回 false 或抛出异常。
    return true;
}
```
```cpp
        client.getClientStub().setCertificateValidationCallback(&schannelValidateCert);
```
RCF client 端也可以配置为向 server 提供一个证书：
```cpp
        RCF::CertificatePtr clientCertPtr( new RCF::PfxCertificate(
            "C:\\clientCert.p12", 
            "password", 
            "CertificateName") );
        client.getClientStub().setCertificate(clientCertPtr);
```
Server 端证书验证与 client 端证书验证的方法相同，但是使用 [RCF::RcfServer::setEnableSchannelCertificateValidation()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#a26480ecccf19f9753c6c53c858e8852d)、[RCF::RcfServer::setCertificateValidationCallback()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#a6fc0948f012a9ad35d409c99f0623887) 和 [RCF::RcfServer::setCaCertificate()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#ab2919c0b64ddb069f1493bfe8238e46f) 函数。

#### 4.2 OpenSSL
当使用基于 `OpenSSL` 的 SSL 传输协议时，需要使用 [RCF::PemCertificate](http://www.deltavsoft.com/doc/class_r_c_f_1_1_pem_certificate.html) 类从 `.pem` 文件加载证书。下面是一个提供 server 证书的例子：
```cpp
        RCF::CertificatePtr serverCertPtr( new RCF::PemCertificate(
            "C:\\serverCert.pem", 
            "password") );
        server.setCertificate(serverCertPtr);
```
Client 可以通过两种方式验证 server 证书。它可以提供一个证书认证中心：
```cpp
        RCF::CertificatePtr caCertPtr( new RCF::PemCertificate(
            "C:\\clientCaCertificate.pem", 
            "password"));
        client.getClientStub().setCaCertificate(caCertPtr);
```
，或者可以在一个回调函数中提供自定义验证逻辑：
```cpp
bool opensslValidateCert(RCF::Certificate * pCert)
{
    RCF::X509Certificate * pX509Cert = static_cast<RCF::X509Certificate *>(pCert);
    if (pX509Cert)
    {
        std::string certName = pX509Cert->getCertificateName();
        std::string issuerName = pX509Cert->getIssuerName();
        X509 * pX509 = pX509Cert->getX509();
        // Custom code to inspect and validate certificate.
        // ...
    }
    // Return true if valid, false if not.
    return true;
}
```
```cpp
        client.getClientStub().setCertificateValidationCallback(&opensslValidateCert);
```
Client 端也可以提供自己的证书，提交给 server:
```cpp
        RCF::CertificatePtr clientCertPtr( new RCF::PemCertificate(
            "C:\\clientCert.pem", 
            "password") );
        client.getClientStub().setCertificate(clientCertPtr);
```
Server 端证书验证与 client 端证书验证的方法相同，但是使用 [RCF::RcfServer::setCertificateValidationCallback()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#a6fc0948f012a9ad35d409c99f0623887) 和 [RCF::RcfServer::setCaCertificate()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_rcf_server.html#ab2919c0b64ddb069f1493bfe8238e46f) 函数。

### 5. 压缩
RCF 支持远程调用的传输级压缩。要构建支持压缩的 RCF，请定义 `RCF_USE_ZLIB`（ 请参阅[构建 RCF](https://love2.io/@lh786020019/doc/RCF-3.1/building_RCF/index.md) ）。

压缩是独立于其他传输协议配置的，使用 [RCF::ClientStub::setEnableCompression()](http://www.deltavsoft.com/doc/class_r_c_f_1_1_client_stub.html#a22f4147ef26b2b117999e1ce7066d953)：
```cpp
        RcfClient<I_PrintService> client(( RCF::TcpEndpoint(port) ));
        client.getClientStub().setEnableCompression(true);
        client.Print("Hello World");
```
RCF 在传输协议阶段之前应用压缩阶段。例如，如果您在同一个连接上配置压缩和 `NTLM`：
```cpp
        RcfClient<I_PrintService> client(( RCF::TcpEndpoint(port) ));
        client.getClientStub().setTransportProtocol(RCF::Tp_Ntlm);      
        client.getClientStub().setEnableCompression(true);
        client.Print("Hello World");
```
，远程调用数据将首先被压缩，然后进行加密并通过网络发送。