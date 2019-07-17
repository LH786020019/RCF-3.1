<!--
 * @Author: haoluo
 * @Date: 2019-07-17 19:32:00
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-17 19:34:13
 * @Description: file content
 -->

## HTTP 和 HTTPS Tunneling
### 1. HTTP 和 HTTPS Tunneling - Server
这个示例演示了一个 server 通过 HTTP 和 HTTPS tunnels 接受 client 连接。
```cpp
#include <iostream>
#include <RCF/RCF.hpp>
#include <RCF/PemCertificate.hpp>
#ifndef RCF_FEATURE_OPENSSL
#error Please define RCF_FEATURE_OPENSSL=1 to build this sample with OpenSSL support.
#endif
// Define RCF interface.
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
class PrintService
{
public:
    void Print(const std::string & msg)
    {
        RCF::RcfSession & session           = RCF::getCurrentRcfSession();
        RCF::TransportType transportType    = session.getTransportType();
        std::cout << std::endl;
        if ( transportType == RCF::Tt_Http )
        {
            std::cout << "Client connecting through a HTTP tunnel." << std::endl;
        }
        else if ( transportType == RCF::Tt_Https )
        {
            std::cout << "Client connecting through a HTTPS tunnel." << std::endl;
        }
        std::cout << "I_PrintService service: " << msg << std::endl;
    }
};
int main()
{
    try
    {
        RCF::RcfInit rcfInit;
        RCF::globals().setDefaultSslImplementation(RCF::Si_OpenSsl);
        
        RCF::RcfServer server;
        // Configure HTTP and HTTPS endpoints.
        server.addEndpoint(RCF::HttpEndpoint("0.0.0.0", 80));
        server.addEndpoint(RCF::HttpsEndpoint("0.0.0.0", 443));
        PrintService printService;
        server.bind<I_PrintService>(printService);
        // Configure server certificate for HTTPS.
        RCF::CertificatePtr serverCertPtr(new RCF::PemCertificate(
            "serverCert.pem",
            "password"));
        server.setCertificate(serverCertPtr);
        server.start();
        std::cout << "Press Enter to exit..." << std::endl;
        std::cin.get();
    }
    catch ( const RCF::Exception & e )
    {
        std::cout << "Error: " << e.getErrorMessage() << std::endl;
    }
    return 0;
}
```

### 2. HTTP 和 HTTPS tunneling - Client
这个示例演示了一个 client 通过 HTTP 和 HTTPS tunnels 连接到 server。
```cpp
#include <iostream>
#include <RCF/RCF.hpp>
#include <RCF/PemCertificate.hpp>
#ifndef RCF_FEATURE_OPENSSL
#error Please define RCF_FEATURE_OPENSSL=1 to build this sample with OpenSSL support.
#endif
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
bool validateCert(RCF::Certificate * pCert)
{
    RCF::X509Certificate * pX509Cert        = dynamic_cast<RCF::X509Certificate *>(pCert);
    if ( pX509Cert )
    {
        std::string certName                = pX509Cert->getCertificateName();
        std::string issuerName              = pX509Cert->getIssuerName();
        X509 * pX509                        = pX509Cert->getX509();
        // Custom code to inspect and validate certificate here.
        // ...
    }
    // Return true if valid, false if not.
    return true;
}
int main()
{
    try
    {
        RCF::RcfInit rcfInit;
        RCF::globals().setDefaultSslImplementation(RCF::Si_OpenSsl);
        // Name of the machine where the server is running.
        std::string serverName = "printsvr";
        // HTTP proxy details (optional).
        std::string proxyName = "web-proxy.acme.com";
        int proxyPort = 8080;
        {
            // Connect to server through a HTTP tunnel.
            
            std::cout << "Connecting through HTTP tunnel." << std::endl;
            RcfClient<I_PrintService> client(RCF::HttpEndpoint(serverName, 80));
            client.getClientStub().setHttpProxy(proxyName);
            client.getClientStub().setHttpProxyPort(proxyPort);
            client.Print("Hello World");
        }
        {
            // Connect to server through a HTTPS tunnel.
            std::cout << "Connecting through HTTPS tunnel." << std::endl;
            RcfClient<I_PrintService> client(RCF::HttpsEndpoint(serverName, 443));
            client.getClientStub().setHttpProxy(proxyName);
            client.getClientStub().setHttpProxyPort(proxyPort);
            // Certificate validation with custom callback.
            client.getClientStub().setCertificateValidationCallback(&validateCert);
            client.Print("Hello World");
        }
    }
    catch ( const RCF::Exception & e )
    {
        std::cout << "Error: " << e.getErrorMessage() << std::endl;
    }
    return 0;
}
```