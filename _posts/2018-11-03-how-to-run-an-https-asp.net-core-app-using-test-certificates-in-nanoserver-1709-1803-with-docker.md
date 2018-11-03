---
layout: post
type : post
title : How to run an Https ASP.NET Core App using test certificates in Nanoserver 1709/1803 with Docker
date: 2018-11-03
tags : [ASP.NET Core, Docker, Https]
published: true
---
Run Https ASP.NET Core Applications in Nanoserver 1709/1803 Containers with Docker

I struggled with this a little bit, so I decided to make the smallest example possible and write the details in here.

### Generating Certificates
First, we need a root certificate that both the server and client are going to trust:
```PowerShell
$rootCert  =  New-SelfSignedCertificate  `
	-Subject "CN=Root CA,OU=Me,O=MyCompany,L=Brussels,ST=Belgium,C=BE"  `
	-CertStoreLocation cert:\CurrentUser\My `
	-Provider "Microsoft Strong Cryptographic Provider"  `
	-DnsName "MyCompany Root CA"  `
	-KeyLength 2048  `
	-KeyAlgorithm "RSA"  `
	-HashAlgorithm "SHA256"  `
	-KeyExportPolicy "Exportable"  `
	-KeyUsage "CertSign",  "CRLSign"
```
The we need to create a ```.cer``` file that will be copied in containers and imported in their trust store:
```PowerShell
Export-Certificate  -Type CERT -Cert $rootCert  -FilePath root-ca.cer
```

The second certificate we need is the one that will be used by the server for TLS and that must be signed by the root certificate:
```PowerShell
$serverCert = New-SelfSignedCertificate  `
	-Subject "CN=server,OU=ESF,O=Ingenico,L=Brussels,ST=Belgium,C=BE"  `
	-CertStoreLocation cert:\CurrentUser\My `
	-Provider "Microsoft Strong Cryptographic Provider"  `
	-Signer $rootCert  `
	-KeyLength 2048  `
	-KeyAlgorithm "RSA"  `
	-HashAlgorithm "SHA256"  `
	-KeyExportPolicy "Exportable"
```

Notice that in this case the certificate is signed by the root certificate. The client will use "server" as the hostname and the certificate has to contain it in order for the client to not reject it because of a miss-match (I'm going on a limb here since I'm no TLS/certificates expert, but after fiddling this is what has been working for me).

This certificate needs to be exported as ```.pfx```:
```PowerShell
Export-PfxCertificate  -Cert $serverCert -Password $password  -FilePath server.pfx
```

### Creating the Server
A tricky bit when using ```1709``` or ```1803``` nanoserver containers is that they don't include PowerShell. You can get it is by getting it from the internet and then copying it in the image (some Microsoft images, such as [the powershell one](https://github.com/PowerShell/PowerShell-Docker/blob/master/release/stable/nanoserver/docker/Dockerfile), do that) but it's a hassle, and actually useless because the PKI module is not part of PowerShell code (pwsh.exe) which means Import-Certificate will not be available anyway.
I found that nanoserver's image contains a tool called ```certoc.exe``` (which is completely undocumented, by the way) that allows you to do that. But surprise, it is not present in the ```1709``` nor ```1803``` images, only in ```sac2016```.
Luckily, ```certoc.exe``` is a standalone tool which can be copied over, so we can solve all this by copying the file from the ```sac2016``` image onto the ```1709``` or ```1803``` image in the ```Dockerfile```:

```Dockerfile
FROM microsoft/nanoserver:sac2016 as tool

FROM microsoft/dotnet:2.1-aspnetcore-runtime-nanoserver-1803

WORKDIR /app
COPY /bin/publish .

COPY --from=tool /Windows/System32/certoc.exe .

USER ContainerAdministrator
RUN certoc.exe -addstore root root-ca.cer
USER ContainerUser

ENTRYPOINT ["dotnet", "MyApp.dll"]
```
Two important things here:
* copy the ```certoc.exe``` file from the tool image into the target image
* import the root certificate (as ContainerAdministrator user)

### Creating the Client

```Dockerfile
FROM microsoft/nanoserver:sac2016 as tool

FROM microsoft/dotnet:2.1-runtime-nanoserver-1803

WORKDIR /app

COPY /bin/publish .

COPY --from=tool /Windows/System32/certoc.exe .
USER ContainerAdministrator
RUN certoc.exe -addstore root root-ca.cer
USER ContainerUser

ENTRYPOINT ["dotnet", "Client.dll"]
```
Again, we copy the tool from the ```sac2016``` image then import the root certificate in the root store of the machine so that it is trusted.

### Full Example
You can find a full example for this, including a client and a server, here: https://github.com/Pvlerick/AspNetCoreTlsInNanoserverDocker

> Written with [StackEdit](https://stackedit.io/).