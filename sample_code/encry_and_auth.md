<!--
 * @Author: haoluo
 * @Date: 2019-07-17 19:18:01
 * @LastEditors: haoluo
 * @LastEditTime: 2019-07-17 19:31:33
 * @Description: file content
 -->
## 加密和认证
### 1. OpenSSL SSL Server
这个示例演示了一个使用 OpenSSL 的 SSL 加密服务器。
```cpp
#include <iostream>
#include <RCF/RCF.hpp>
#include <RCF/PemCertificate.hpp>
#ifndef RCF_FEATURE_OPENSSL
#error Please define RCF_FEATURE_OPENSSL=1 to build this sample with OpenSSL support.
#endif
// 定义 RCF 接口
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
class PrintService{
public:
    void Print(const std::string & msg){
        std::cout << "I_PrintService service: " << msg << std::endl;
    }
};
int main(){
    try{
        RCF::RcfInit rcfInit;
        RCF::globals().setDefaultSslImplementation(RCF::Si_OpenSsl);
        
        RCF::RcfServer server(RCF::TcpEndpoint("127.0.0.1", 50001));
        PrintService printService;
        server.bind<I_PrintService>(printService);
        // 加载 server 证书
        RCF::CertificatePtr serverCertPtr(new RCF::PemCertificate(
            "serverCert.pem",
            "password"));
        server.setCertificate(serverCertPtr);
        // 要求所有 client 都使用 SSL
        std::vector<RCF::TransportProtocol> supportedProtocols = { RCF::Tp_Ssl };
        server.setSupportedTransportProtocols(supportedProtocols);
        server.start();
        std::cout << "Press Enter to exit..." << std::endl;
        std::cin.get();
    } catch ( const RCF::Exception & e ) {
        std::cout << "Error: " << e.getErrorMessage() << std::endl;
    }
    return 0;
}
```

### 2. OpenSSL SSL Client
这个示例演示了一个使用 OpenSSL 的 SSL 加密 client。
```cpp
#include <iostream>
#include <RCF/RCF.hpp>
#include <RCF/PemCertificate.hpp>
#ifndef RCF_FEATURE_OPENSSL
#error Please define RCF_FEATURE_OPENSSL=1 to build this sample with OpenSSL support.
#endif
// 定义 RCF 接口
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
// 回调函数，用于验证 server 提供的证书。
bool validateServerCertificate(RCF::Certificate * pCert){
    RCF::X509Certificate * pX509Cert    = dynamic_cast<RCF::X509Certificate *>(pCert);
    if ( pX509Cert ) {
        std::string certName            = pX509Cert->getCertificateName();
        std::string issuerName          = pX509Cert->getIssuerName();
        X509 * pX509                    = pX509Cert->getX509();
        
        // 这里检查和验证证书的自定义代码。
        // ...
        std::cout << "Server certificate name: " << certName << std::endl;
        std::cout << "Server certificate issuer: " << issuerName << std::endl;
    }
    // 如果有效返回 true，如果无效返回 false。
    return true;
}
int main(){
    try{
        RCF::RcfInit rcfInit;
        RCF::globals().setDefaultSslImplementation(RCF::Si_OpenSsl);
        RcfClient<I_PrintService> client(RCF::TcpEndpoint("127.0.0.1", 50001));
        
        client.getClientStub().setTransportProtocol(RCF::Tp_Ssl);
        // 配置 server 证书验证。
        bool useCertificateAuthority = false;
        bool useCustomCallback = true;
        if ( useCertificateAuthority ) {
            // 使用 CertificateAuthority 进行证书验证。
            // 加载一个 CertificateAuthority certificate。
            RCF::CertificatePtr caCertPtr(new RCF::PemCertificate(
                "clientCaCertificate.pem",
                "password"));
            client.getClientStub().setCaCertificate(caCertPtr);
        } else if ( useCustomCallback ) {
            // 使用自定义回调进行证书验证。
            client.getClientStub().setCertificateValidationCallback(&validateServerCertificate);
        }
        // 配置一个 client 证书(可选)。
        RCF::CertificatePtr clientCertPtr(new RCF::PemCertificate(
            "clientCert.pem",
            "password"));
        client.getClientStub().setCertificate(clientCertPtr);
        
        client.Print("Hello World");
    } catch ( const RCF::Exception & e ) {
        std::cout << "Error: " << e.getErrorMessage() << std::endl;
    }
    return 0;
}
```

### 3. Schannel SSL Server
这个示例演示了一个使用 `Schannel` 的 SSL 加密 server。这段代码只能在 Windows 上运行。
```cpp
#include <iostream>
#include <RCF/RCF.hpp>
#include <RCF/Win32Certificate.hpp>
#ifndef RCF_WINDOWS
#error This sample requires Schannel and can only be built on Windows.
#endif
// Define RCF interface.
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
class PrintService{
public:
    void Print(const std::string & msg){
        std::cout << "I_PrintService service: " << msg << std::endl;
    }
};
int main(){
    try{
        RCF::RcfInit rcfInit;
        RCF::globals().setDefaultSslImplementation(RCF::Si_Schannel);
        
        RCF::RcfServer server(RCF::TcpEndpoint("127.0.0.1", 50001));
        PrintService printService;
        server.bind<I_PrintService>(printService);
        // Load server certificate.
        bool loadCertFromFile = true;
        bool loadCertFromStore = false;
        if ( loadCertFromFile ) {
            // Load server certificate from a file.
            RCF::CertificatePtr serverCertPtr(new RCF::PfxCertificate(
                "serverCert.p12",
                "password",
                "CertificateName"));
            server.setCertificate(serverCertPtr);
        } else if ( loadCertFromStore ) {
            // Load server certificate from a Windows certificate store.
            RCF::CertificatePtr serverCertPtr(new RCF::StoreCertificate(
                RCF::Cl_LocalMachine,
                RCF::Cs_My,
                "CertificateName"));
            server.setCertificate(serverCertPtr);
        }
        // Require all clients to use SSL.
        std::vector<RCF::TransportProtocol> supportedProtocols = { RCF::Tp_Ssl };
        server.setSupportedTransportProtocols(supportedProtocols);
        server.start();
        std::cout << "Press Enter to exit..." << std::endl;
        std::cin.get();
    } catch ( const RCF::Exception & e ) {
        std::cout << "Error: " << e.getErrorMessage() << std::endl;
    }
    return 0;
}
```

### 4. Schannel SSL Client
这个示例演示了一个使用 `Schannel` 的 SSL 加密 client。这段代码只能在 Windows 上运行。
```cpp
#include <iostream>
#include <RCF/RCF.hpp>
#include <RCF/Win32Certificate.hpp>
#ifndef RCF_WINDOWS
#error This sample requires Schannel and can only be built on Windows.
#endif
// Define RCF interface.
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
#include <iostream>
#include <RCF/RCF.hpp>
// Callback function to validate the certificate the server presents.
bool validateServerCertificate(RCF::Certificate * pCert){
    RCF::Win32Certificate * pWin32Cert  = dynamic_cast<RCF::Win32Certificate *>(pCert);
    if ( pWin32Cert ) {
        std::string certName            = pWin32Cert->getCertificateName();
        std::string issuerName          = pWin32Cert->getIssuerName();
        PCCERT_CONTEXT pCertCtx         = pWin32Cert->getWin32Context();
        // Custom code to inspect and validate certificate here.
        // ...
        std::cout << "Server certificate name: " << certName << std::endl;
        std::cout << "Server certificate issuer: " << issuerName << std::endl;
    }
    // Return true if valid, false if not.
    return true;
}
int main(){
    try{
        RCF::RcfInit rcfInit;
        RCF::globals().setDefaultSslImplementation(RCF::Si_Schannel);
        RcfClient<I_PrintService> client(RCF::TcpEndpoint("127.0.0.1", 50001));
        
        client.getClientStub().setTransportProtocol(RCF::Tp_Ssl);
        // Configure server certificate validation.
        bool useCertificateAuthority = false;
        bool useCustomCallback = true;
        bool useSchannelValidation = false;
        if ( useCertificateAuthority ) {
            // Certificate validation with certificate authority.
            RCF::CertificatePtr caCertPtr(new RCF::PfxCertificate(
                "C:\\clientCaCertificate.p12",
                "password",
                "CaCertificatename"));
            client.getClientStub().setCaCertificate(caCertPtr);
        } else if ( useCustomCallback ) {
            // Certificate validation with custom callback.
            client.getClientStub().setCertificateValidationCallback(&validateServerCertificate);
        } else if ( useSchannelValidation ) {
            // Certificate validation using built-in Schannel validation.
            client.getClientStub().setEnableSchannelCertificateValidation("CertificateName");
        }
         
        client.Print("Hello World");
    } catch ( const RCF::Exception & e ) {
        std::cout << "Error: " << e.getErrorMessage() << std::endl;
    }
    return 0;
}
```

### 5. Kerberos 和 NTLM Server
这个示例演示了一个 `Kerberos` 和 `Kerberos` 加密 server。这段代码只能在 Windows 上运行。
```cpp
#include <iostream>
#include <RCF/RCF.hpp>
#ifndef RCF_WINDOWS
#error This sample requires Kerberos and NTLM SSPI providers and can only be built on Windows.
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
        RCF::RcfSession & session                   = RCF::getCurrentRcfSession();
        RCF::TransportProtocol transportProtocol    = session.getTransportProtocol();
        // Determine which encryption protocol the client is using.
        std::cout << std::endl;
        if ( transportProtocol == RCF::Tp_Ntlm )
        {
            std::cout << "Client connection authenticated and encrypted using NTLM." << std::endl;
        }
        else if ( transportProtocol == RCF::Tp_Kerberos )
        {
            std::cout << "Client connection authenticated and encrypted using Kerberos." << std::endl;
        }
        else if ( transportProtocol == RCF::Tp_Negotiate )
        {
            std::cout << "Client connection authenticated and encrypted using Negotiate." << std::endl;
        }
        // Determine the user name the client was authenticated as.
        std::cout << "Client user name: " << session.getClientUserName() << std::endl;
        std::cout << "I_PrintService service: " << msg << std::endl;
    }
};
int main()
{
    try
    {
        RCF::RcfInit rcfInit;
        
        RCF::RcfServer server(RCF::TcpEndpoint("127.0.0.1", 50001));
        PrintService printService;
        server.bind<I_PrintService>(printService);
        // Require all clients to use NTLM, Kerberos or Negotiate.
        std::vector<RCF::TransportProtocol> supportedProtocols = { RCF::Tp_Ntlm, RCF::Tp_Kerberos, RCF::Tp_Negotiate };
        server.setSupportedTransportProtocols(supportedProtocols);
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

### 6. Kerberos 和 NTLM Client
这个示例演示了一个 `Kerberos` 和 `Kerberos` 加密 client。这段代码只能在 Windows 上运行。
```cpp
#include <iostream>
#include <RCF/RCF.hpp>
// Define RCF interface.
RCF_BEGIN(I_PrintService, "I_PrintService")
    RCF_METHOD_V1(void, Print, const std::string &)
RCF_END(I_PrintService)
int main()
{
    try
    {
        RCF::RcfInit rcfInit;
        RcfClient<I_PrintService> client(RCF::TcpEndpoint("127.0.0.1", 50001));
        // Configure explicit client credentials.
        bool useExplicitCreds = false;
        std::string explicitUserName = "Domain\\JoeBloggs";
        std::string explicitPassword = "MyPassword";
        if ( useExplicitCreds )
        {
            client.getClientStub().setUserName(explicitUserName);
            client.getClientStub().setPassword(explicitPassword);
        }
        else
        {
            // If explicit credentials are not specified, the client will pick up 
            // the credentials of the currently logged on user.
        }
        // Connect using NTLM.
        std::cout << "Connecting using NTLM." << std::endl;
        client.getClientStub().setTransportProtocol(RCF::Tp_Ntlm);
        client.Print("Hello World");
        
        // Connect using Kerberos.
        std::cout << "Connecting using Kerberos." << std::endl;
        client.getClientStub().setTransportProtocol(RCF::Tp_Kerberos);
        client.getClientStub().setKerberosSpn("Domain\\ServerAccount");
        client.Print("Hello World");
        
        // Connect using Negotiate.
        std::cout << "Connecting using Negotiate." << std::endl;
        client.getClientStub().setTransportProtocol(RCF::Tp_Negotiate);
        client.getClientStub().setKerberosSpn("Domain\\ServerAccount");
        client.Print("Hello World");
    }
    catch ( const RCF::Exception & e )
    {
        std::cout << "Error: " << e.getErrorMessage() << std::endl;
    }
    return 0;
}
```