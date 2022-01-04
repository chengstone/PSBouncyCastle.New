## 证书制作：  
[PowerShell脚本创建证书](https://stackoverflow.com/questions/44004052/generate-self-signed-certificate-with-root-ca-signer)  

1. 打开Power Shell  
2. 引入制作证书的模块：  
```
Import-Module .\PSBouncyCastle.New\PSBouncyCastle.psm1  
```
3. 制作根证书：  
```
$RootCA = New-SelfSignedCertificate2 -subjectName "CN=RootCA" -IsCA $true -FriendlyName "RootCA"  
```
4. 导出根证书：  
```
Export-Certificate -Certificate $RootCA -OutputFile "RootCA.pfx" -X509ContentType Pfx
```
5. 通过根证书，制作用于【客户端身份验证】的个人证书：  
```
$ClientCert = New-SelfSignedCertificate2 -subjectName "CN=ClientCertByRoot" -issuer $RootCA -EKU @{ "ClientAuthentication" = $true } -FriendlyName "ClientCertByRoot"  
```
6. 导出个人证书：
```
Export-Certificate -Certificate $ClientCert -OutputFile "ClientCertByRoot.pfx" -X509ContentType Pfx
```


## 个人证书必须含有的字段：  
1. 增强型密钥用法  
例：  
客户端身份验证 (1.3.6.1.5.5.7.3.2)  
2. 授权密钥标识符  
本字段必须含有Certificate SerialNumber，该值等于根证书的序列号（SerialNumber）。  
例：  
```
KeyID=f1d3a4ff7f505e49fc6723d10311821945ac6ce2  
Certificate Issuer:  
     Directory Address:  
          CN=RootCA  
Certificate SerialNumber=147437785926fca9  
```


## 服务端的准备工作  
1. 安装根证书到【受信任的根证书颁发机构】  
2. 安装个人证书到【个人】  
3. 添加SSL证书用于https端口监听：  
```
netsh http add sslcert ipport=0.0.0.0:8443 `
      certhash=3ac6c8655e583ad73bb0eeded377708051fabe0b `
      appid=`{BAA299E7-BF85-4538-B7EF-12E309D67FFC`}
```
其中certhash是个人证书的指纹（thumbprint）  
appid是GUID，可以使用任何GUID生成器生成，也可以用 Visual Sudio IDE 內建的小工具生成（工具(T)->创建GUID(G)）  
命令行提示【成功添加 SSL 证书】，即表示成功  
其他命令：  
```  
删除SSL证书：  
netsh http delete sslcert ipport=0.0.0.0:8443  
查看个人证书：   
certutil -store My  
修复个人证书：  
certutil -repairstore My 证书号  
```  


## 获得Windows系统管理员权限（TrustedInstaller）  
1. 打开Power Shell  
2. 输入如下命令：  
```  
Save-Module -Name NtObjectManager -Path D:\token
Install-Module -Name NtObjectManager
Set-ExecutionPolicy Unrestricted
Import-Module NtObjectManager
sc.exe start TrustedInstaller
Set-NtTokenPrivilege SeDebugPrivilege
$p = Get-NtProcess -Name TrustedInstaller.exe
$proc = New-Win32Process cmd.exe -CreationFlags NewConsole -ParentProcess $p
```  
3. 在打开的cmd中输入命令查看权限： 
```   
whoami /groups /fo list
```  
4. 键入mmc即可打开控制台管理证书  


## 在Postman中设置个人证书  
1. settings->General->Request->SSL certificate verification: turn Off  
2. settings->Certificates:  
press Add Certificate   
Host: localhost:8443  
PFX file：X:/path/to/ClientCertByRoot.pfx  
3. press Add  


## 制作证书备用方案  
[制作证书用于https监听](https://stackoverflow.com/questions/11403333/httplistener-with-https-support/11457719#11457719)  
制作根证书：  
```  
makecert -n "CN=vMargeCA" -r -sv vMargeCA.pvk vMargeCA.cer  
```  

制作个人证书：  
```  
makecert -sk vMargeSignedByCA -iv vMargeCA.pvk -n "CN=vMargeSignedByCA" -ic vMargeCA.cer vMargeSignedByCA.cer -sr localmachine -ss My  
```  

用于客户端身份验证的命令参数：  
```  
-eku 1.3.6.1.5.5.7.3.2 
```  

Bind certificate to IP address:port and application.  
```  
netsh http add sslcert ipport=0.0.0.0:8443 certhash=585947f104b5bce53239f02d1c6fed06832f47dc appid={df8c8073-5a4b-4810-b469-5975a9c95230}
```  

[在Self Host启用SSL](https://dotblogs.com.tw/yc421206/2020/02/04/enable_ssl_on_self_host_nancy_owin)  
[制作用于客户端身份验证的证书](https://github.com/OpenVPN/easy-rsa/blob/master/README.quickstart.md)  
Setup and signing the first request  
-----------------------------------  

Here is a quick run-though of what needs to happen to start a new PKI and sign  
your first entity certificate:  

1. Choose a system to act as your CA and create a new PKI and CA:  
```  
        ./easyrsa init-pki
        ./easyrsa build-ca
```  
2. On the system that is requesting a certificate, init its own PKI and generate  
   a keypair/request. Note that init-pki is used _only_ when this is done on a  
   separate system (or at least a separate PKI dir.) This is the recommended  
   procedure. If you are not using this recommended procedure, skip the next  
   import-req step.  
```  
        ./easyrsa init-pki
        ./easyrsa gen-req EntityName
```  
3. Transport the request (.req file) to the CA system and import it. The name  
   given here is arbitrary and only used to name the request file.  
```  
        ./easyrsa import-req /tmp/path/to/import.req EntityName
```  
4. Sign the request as the correct type. This example uses a client type:  
```  
        ./easyrsa sign-req client EntityName
```  
5. Transport the newly signed certificate to the requesting entity. This entity  
   may also need the CA cert (ca.crt) unless it had a prior copy.  

6. The entity now has its own keypair, signed cert, and the CA.  

ca.crt：根证书   
private/ca.key：根证书私钥  
issued/EntityName.crt：个人证书  
private/EntityName.key：个人证书私钥  

[New-SelfSignedCertificate命令创建自签名证书](https://github.com/ssh382736/-Kubernetes-poc/wiki/Azure-Vpn-Gateway%E8%87%AA%E7%AD%BE%E5%90%8D%E8%AF%81%E4%B9%A6%E5%8F%82%E6%95%B0)
```  
1.生成一个服务器证书

$cert = New-SelfSignedCertificate -Type Custom -KeySpec Signature -Subject "CN=P2SRootCert" -KeyExportPolicy Exportable -HashAlgorithm sha256 -KeyLength 2048 ` -CertStoreLocation "Cert:\LocalMachine\My" -KeyUsageProperty Sign -KeyUsage CertSign

2.生成一个客户端证书

New-SelfSignedCertificate -Type Custom -KeySpec Signature -Subject "CN=P2SChildCert" -KeyExportPolicy Exportable -HashAlgorithm sha256 -KeyLength 2048 -CertStoreLocation "Cert:\LocalMachine\My" -Signer $cert -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.2")
```  

[其他power shell脚本](https://www.powershellgallery.com/packages/SelfSignedCertificate/0.0.4/Content/SelfSignedCertificate.psm1)

其他命令：  
```  
pvk2pfx -pvk vMargeSignedByCA4.pvk -spc vMargeSignedByCA4.cer -pfx vMargeSignedByCA4.pfx
openssl rsa -inform pvk -in vMargeSignedByCA2.pvk -outform pem -out vMargeSignedByCA2.pem
openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt -in vMargeSignedByCA2.pem -out vMargeSignedByCA2.key
openssl pkcs8 -topk8 -inform PEM -outform PEM -in vMargeSignedByCA2.pem -out vMargeSignedByCA2-new.key
```  


PSBouncyCastle.New
==================

**PSBouncyCastle** is a PowerShell module that allows you to use the
crypto functionality from the [Legion of the BouncyCastle](http://www.bouncycastle.org/)
.NET libraries.

Currently it covers the X509 certificate functionality, in particular
allowing you to replace `makecert.exe` (from the Windows SDK) with
native PowerShell cmdlets.

This was written by *RLipscome* and forked to the current version in order to update the Crypto library.

Installation
--

	Set-Location (Join-Path (Split-Path $PROFILE) 'Modules')
	git clone https://github.com/LimpingNinja/PSBouncyCastle.New.git
	Import-Module PSBouncyCastle

*Note:* I'll get this listed on [PsGet](http://psget.net/) at some point.
