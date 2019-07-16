<!--
 * @Author: haoluo
 * @Date: 2019-07-16 09:18:41
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-16 10:03:13
 * @Description: file content
 -->
## 传输协议
传输协议用于转换通过传输的数据。RCF使用传输协议为远程调用提供身份验证、加密和压缩。

RCF目前支持NTLM、Kerberos、协商和SSL传输协议。NTLM、Kerberos和协商只在Windows平台上受支持，而SSL在所有平台上都受支持。

此外，RCF还支持基于zlib的远程调用压缩。

传输协议是通过调用RCF::ClientStub::setTransportProtocol()在客户端连接上配置的：
```cpp
        RcfClient<I_PrintService> client(( RCF::TcpEndpoint(port) ));
        client.getClientStub().setTransportProtocol(RCF::Tp_Ntlm);
```
在客户机连接的服务器会话中，可以调用RCF::RcfSession::getTransportProtocol()来确定客户机正在使用的传输prococol：
```cpp
        RCF::RcfSession & session = RCF::getCurrentRcfSession();
        RCF::TransportProtocol tp = session.getTransportProtocol();
```
NTLM
要在客户端连接上配置NTLM：
```cpp
    RcfClient<I_PrintService> client(( RCF::TcpEndpoint(port) ));
    client.getClientStub().setTransportProtocol(RCF::Tp_Ntlm);
    client.Print("Hello World");
```
在服务器端，您可以确定客户端的Windows用户名，并模拟它们：
```cpp
        std::string clientUsername = session.getClientUserName();
        RCF::SspiImpersonator impersonator(session);
        // We are now impersonating the client, until impersonator goes out of scope.
        // ...
```
Kerberos
要在客户机连接上配置Kerberos：
```cpp
        client.getClientStub().setTransportProtocol(RCF::Tp_Kerberos);
        client.getClientStub().setKerberosSpn("Domain\\ServerAccount");
        client.Print("Hello World");
```
注意，客户机需要调用ClientStub::setKerberosSpn()来指定它期望服务器运行的用户名。这称为服务器的SPN(服务主体名)，Kerberos协议中使用它来实现相互身份验证。如果服务器不在此帐户下运行，连接将失败。

在服务器端，您可以确定客户端的Windows用户名，并模拟它们：
```cpp
        std::string clientUsername = session.getClientUserName();
        RCF::SspiImpersonator impersonator(session);
        // We are now impersonating the client, until impersonator goes out of scope.
        // ...
```
Negotiate
协商是NTLM和Kerberos协议之间的协商协议。如果可能，它将解析为Kerberos，否则返回到NTLM。

配置客户端连接上的协商：
```cpp
        client.getClientStub().setTransportProtocol(RCF::Tp_Negotiate);
        client.getClientStub().setKerberosSpn("Domain\\ServerAccount");
        client.Print("Hello World");
```
与Kerberos传输协议一样，您需要为服务器提供SPN。

在服务器端，您可以确定客户端的Windows用户名，并模拟它们：
```cpp
        std::string clientUsername = session.getClientUserName();
        RCF::SspiImpersonator impersonator(session);
        // We are now impersonating the client, until impersonator goes out of scope.
        // ...
```
SSL
RCF提供了两种SSL传输协议实现。一个基于跨平台OpenSSL库，另一个基于仅限windows的Schannel包。

只有在定义了RCF_USE_OPENSSL之后，才可以使用OpenSSL支持。

Schannel支持只在Windows构建中可用，不需要任何定义。

如果在Windows构建中定义RCF_USE_OPENSSL, RCF将使用OpenSSL而不是Schannel。你想应该使用Schannel,尽管定义RCF_USE_OPENSSL,您可以设置SSL实现单个服务器和客户端使用RCF:: RcfServer: setSslImplementation()和RCF:: ClientStub: setSslImplementation()函数,或者将其设置为整个RCF运行时使用RCF::全局变量:setDefaultSslImplementation()函数。

SSL协议使用证书对客户机的服务器进行身份验证(也可以选择对服务器的客户机进行身份验证)。证书和证书验证功能由两种SSL传输协议实现以不同的方式处理，如下所述。

Schannel
要配置服务器以接受SSL连接，需要提供服务器证书。RCF提供了RCF::PfxCertificate类，用于从.pfx和.p12文件加载证书：
```cpp
        RCF::CertificatePtr serverCertPtr( new RCF::PfxCertificate(
            "C:\\serverCert.p12", 
            "password", 
            "CertificateName") );
        server.setCertificate(serverCertPtr);
```
RCF还提供了RCF::StoreCertificate类，用于从Windows证书商店加载证书：
```cpp
        RCF::CertificatePtr serverCertPtr( new RCF::StoreCertificate(
            RCF::Cl_LocalMachine,
            RCF::Cs_My,
            "CertificateName") );
        server.setCertificate(serverCertPtr);
```
在客户机上，需要提供验证服务器提供的证书的方法。有几种方法可以做到这一点。您可以让Schannel包应用它自己的内部验证逻辑，它将遵从本地安装的证书颁发机构：
```cpp
        client.getClientStub().setTransportProtocol(RCF::Tp_Ssl);
        client.getClientStub().setEnableSchannelCertificateValidation("CertificateName");
        client.Print("Hello World");
```
您还可以自己提供一个特定的证书颁发机构，用于验证服务器证书：
```cpp
        RCF::CertificatePtr caCertPtr( new RCF::PfxCertificate(
            "C:\\clientCaCertificate.p12", 
            "password", 
            "CaCertificatename"));
        client.getClientStub().setCaCertificate(caCertPtr);
```
您还可以编写自己的自定义证书验证逻辑：
```cpp
bool schannelValidateCert(RCF::Certificate * pCert)
{
    RCF::Win32Certificate * pWin32Cert = static_cast<RCF::Win32Certificate *>(pCert);
    if (pWin32Cert)
    {
        RCF::tstring certName = pWin32Cert->getCertificateName();
        RCF::tstring issuerName = pWin32Cert->getIssuerName();
        PCCERT_CONTEXT pContext = pWin32Cert->getWin32Context();
        // Custom code to inspect and validate certificate.
        // ...
    }
    // Return true if the certificate is considered valid. Otherwise, return false,
    // or throw an exception.
    return true;
}
```
```cpp
        client.getClientStub().setCertificateValidationCallback(&schannelValidateCert);
```
RCF客户端也可以配置为向服务器提供证书：
```cpp
        RCF::CertificatePtr clientCertPtr( new RCF::PfxCertificate(
            "C:\\clientCert.p12", 
            "password", 
            "CertificateName") );
        client.getClientStub().setCertificate(clientCertPtr);
```
服务器端证书验证与客户端证书验证的方法相同，但是使用RCF::RcfServer::setEnableSchannelCertificateValidation()、RCF::RcfServer::setCertificateValidationCallback()和RCF::RcfServer::setCaCertificate()函数。

OpenSSL
当使用基于openssl的SSL传输协议时，需要使用RCF::PemCertificate类从.pem文件加载证书。下面是一个提供服务器证书的例子：
```cpp
        RCF::CertificatePtr serverCertPtr( new RCF::PemCertificate(
            "C:\\serverCert.pem", 
            "password") );
        server.setCertificate(serverCertPtr);
```
客户机可以通过两种方式验证服务器证书。它可以提供一个证书颁发机构证书：
```cpp
        RCF::CertificatePtr caCertPtr( new RCF::PemCertificate(
            "C:\\clientCaCertificate.pem", 
            "password"));
        client.getClientStub().setCaCertificate(caCertPtr);
```
，或者可以在回调函数中提供自定义验证逻辑：
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
客户端也可以提供自己的证书，提交给服务器:
```cpp
        RCF::CertificatePtr clientCertPtr( new RCF::PemCertificate(
            "C:\\clientCert.pem", 
            "password") );
        client.getClientStub().setCertificate(clientCertPtr);
```
服务器端证书验证与客户端证书验证的方法相同，但是使用RCF::RcfServer::setCertificateValidationCallback()和RCF::RcfServer::setCaCertificate()函数。

压缩
RCF支持远程调用的传输级压缩。要构建支持压缩的RCF，请定义RCF_USE_ZLIB(参见构建RCF)。

压缩是独立于其他传输协议配置的，使用RCF::ClientStub::setEnableCompression()：
```cpp
        RcfClient<I_PrintService> client(( RCF::TcpEndpoint(port) ));
        client.getClientStub().setEnableCompression(true);
        client.Print("Hello World");
```
RCF在传输协议阶段之前应用压缩阶段。例如，如果您在同一个连接上配置压缩和NTLM：
```cpp
        RcfClient<I_PrintService> client(( RCF::TcpEndpoint(port) ));
        client.getClientStub().setTransportProtocol(RCF::Tp_Ntlm);      
        client.getClientStub().setEnableCompression(true);
        client.Print("Hello World");
```
，远程调用数据将首先被压缩，然后进行加密并通过网络发送。