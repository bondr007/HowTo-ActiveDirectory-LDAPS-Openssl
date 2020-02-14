Skip ahead to [Setup LDAPS using self-signed cert made with openssl](#setup-ldaps-using-self-signed-cert-made-with-openssl) if you do not need any background information.
Also,check out [my accompanying github repo](https://github.com/bondr007/HowTo-ActiveDirectory-LDAPS-Openssl) which contains all the files used in this guide. Inside, see [just_the_commands.md](https://github.com/bondr007/HowTo-ActiveDirectory-LDAPS-Openssl) to quickly run through just the commands.

## Insecure LDAP is dying, Long Live Secure LDAPS
Microsoft will begin enforcing secure connections for Active Directory LDAP ~~in [March of 2020](https://support.microsoft.com/en-us/help/4520412/2020-ldap-channel-binding-and-ldap-signing-requirement-for-windows).~~ Update: Microsoft has extended the deadline to "[2nd half of 2020](https://portal.msrc.microsoft.com/en-us/security-guidance/advisory/ADV190023)". This is the third extension Microsoft has made since first announcing this would be coming back in 2017. Active Directory has long been a haven of questionable security. Microsoft has made several great improvements for security in recent years and this most recent change is designed to plug one of the long-lived security weaknesses of Active Directory.

## Why is it needed
Most Active Directory traffic is sent over plain text insecure LDAP protocol on port 389 for authentication and queries. Active Directory joined machines authenticate using windows integrated authentication which uses encrypted methods such as kerberos or NTLM. In the same way that plain text HTTP is insecure LDAP is also vulnerable to man-in-the-middle attacks, and exposure of sensitive information such as username/passwords. LDAPS like HTTPS transmits its data over an encrypted tunnel using SSL or TLS. 

## How it works
For Active Directory to use LDAPS just like a webserver using HTTPS it needs a certificate issued to it and installed. If you are familiar with certs for webservers the process follows the same way. Create a CSR, certificate signing request, send that to a CA, and then install the client certificate created from the CA. [Here is a great article by cloudflare about SSL/TLS and certs](https://www.cloudflare.com/learning/ssl/how-does-ssl-work/)

## Self-signed or public CA.
Publicly signed certs are often already trusted by many services, but are not free for a cert that has an validity period greater a few months. For most systems connecting using LDAPS this benefit of a cert from a public CA is mute since they have a separate truststore just for LDAPS that typically does not contain any public CAs. So for most Self-signed is the way to go, since it is free and your can set extremely long expiration dates.

## Why not Microsoft CA (Certificate Authority) Server
When initially looking to configure LDAPS for AD I looked into creating a Microsoft CA server. I ran into several limitations for my use case. First off I found Microsoft's documentation to be quite long and unnecessarily confusing. Once I figured it all out it is not too bad, but as you will see the openssl route is quite a bit easier as long as it fits your use case. The primary reason to use Microsoft CA Server is if you plan on issuing certs for other internal only services like internal webservers. Due to the abundance of methods to get free publicly signed certs like [Let’s Encrypt](https://letsencrypt.org/) for webservers I prefer to use a publicly signed cert even for internal webservers.

# See if your application is using plain text LDAP
From the server running your application you can look at the outbound network traffic and check if there is anything communicating to one of your AD Domain Controllers IP addresses over the default LDAP port of 389. LDAPS uses port 636. The netstat command can be used on both linux and windows to see your open network connections.
### Find connections on port 389: Linux
```bash
foo@bar:~$ netstat -antlp | grep 389 | grep ESTABLISHED
tcp        0      0 127.0.0.1:46046         192.168.1.10:389        ESTABLISHED -
tcp        0      0 127.0.0.1:34389         216.58.194.78:443       ESTABLISHED -
```
We can see that this machine is communicating to port 389 on the ip 192.168.1.10 which is a AD Domain controller in my test environment.

### find connections on port 389: Windows
```
C:\Windows\system32>netstat -ant | findstr 389 | findstr ESTABLISHED
  TCP    127.0.0.1:46046        192.168.1.10:389       ESTABLISHED     InHost
  TCP    127.0.0.1:43894        10.2.212.20:64284      ESTABLISHED     InHost
```
Again we see 192.168.1.10:389 which indicates a program connecting to a AD controller using LDAP on port 389

# Setup LDAPS using self-signed cert made with openssl
### Prerequisites
* openssl
* Need to know:
 - your active directory domain name. ex: __example.com__
 - your active directory domain controller's name. ex: __ad01.example.com__

Here is how to install openssl if you do not already have it:
```
#For Debian/Ubuntu 
sudo apt-get install openssl
#For rhel/centos
sudo yum -y install openssl
```
It is also possible to install it on windows. See this guide for installing openssl on windows: https://tecadmin.net/install-openssl-on-windows/

### Creating the CA (Certificate Authority)
Create a directory for us to work in. *pro tip: make your life easy and mount a directory on your AD controller from the machine with openssl. We will need to move a few files back and forth, mounting it over smb makes this easy*

created a text file named ca_san.conf with the following contents, modifying as needed. ex: "example.com" to your domain.
```conf
#ca_san.conf
[ req ]
distinguished_name = req_distinguished_name
req_extensions     = v3_ca

[ req_distinguished_name ]
# Descriptions
countryName=Country Name (2 letter code)
stateOrProvinceName=State or Province Name (full name)
localityName=Locality Name (eg, city)
0.organizationName=Your Company/Organization Name.
1.organizationName=Organizational Unit Name (Department)
commonName=Your Domain Name

#Modify for your details here or answer the prompts from openssl
countryName_default=US
stateOrProvinceName_default=Texas
localityName_default=Dallas
0.organizationName_default=My Company Name LTD.
1.organizationName_default=IT
commonName_default=example.com
[ v3_ca ]
keyUsage=critical,keyCertSign
basicConstraints=critical,CA:TRUE,pathlen:1
extendedKeyUsage=serverAuth
subjectAltName = @alt_names
#Modify for your details. Must include the commonName in the list below also. 
#The *.example.com will allow all Domain controllers with the hostname somthing.example.com to use the cert.
[alt_names]
DNS.1 = *.example.com
DNS.2 = example.com
```
Next save that file to a directory named LDAPS
```bash
foo@bar:~$ mkdir LDAPS && cd LDAPS

# generate the ca key, create a password and keep it for use throughout this guide.
foo@bar:~/LDAPS$ openssl genrsa -des3 -out ca.key 4096
Generating RSA private key, 4096 bit long modulus (2 primes)
...........++++
.............................................................................................++++
Enter pass phrase for ca.key:
Verifying - Enter pass phrase for ca.key:

# create ca cert with valid of 10 years with info based off the provided ca_san.conf file, it will prompt for the password we created earlier 
foo@bar:~/LDAPS$ openssl req -new -x509 \
    -extensions v3_ca \
    -days 3650 \
    -key ca.key \
    -out ca.crt \
    -config ca_san.conf

foo@bar:~/LDAPS$ ls
ca.crt  ca.key
```
Now we have created two files. ca.key and ca.crt

Next we will add the ca.crt as a Trusted Root Certificate and create a Certificate Signing request(CSR) on an AD controller

In powershell __as Admin__ on an AD controller copy over the ca.crt file and run the following to import it as a Trusted Root Certificate
```powershell
#import the cert as a trusted CA on the domain controller
Import-Certificate -FilePath ca.crt  -CertStoreLocation 'Cert:\LocalMachine\Root' -Verbose
```
Create a text file named request.inf with the following contents edited for your environment
```ini
;----------------- request.inf -----------------
[Version]
 Signature="$Windows NT$"

;The Subject will need to be your active directory domain name
[NewRequest]
 Subject = "CN=example.com"
 KeySpec = 1
 KeyLength = 4096
 Exportable = TRUE
 MachineKeySet = TRUE
 SMIME = FALSE
 PrivateKeyArchive = FALSE
 UserProtected = FALSE
 UseExistingKeySet = FALSE
 ProviderName = "Microsoft RSA SChannel Cryptographic Provider"
 ProviderType = 12
 RequestType = PKCS10
 KeyUsage = 0xa0

[EnhancedKeyUsageExtension]
 OID = 1.3.6.1.5.5.7.3.1 ; Server Authentication
;The following will add a subject alternative name of a wildcard cert on *.example.com
;so any ad controller with a hostname of somththing.example.com can use it.
[Extensions]
2.5.29.17 = "{text}"
_continue_ = "dns=*.example.com&"
_continue_ = "dns=example.com&"
```
Next on the AD controller run certreq passing in the request.inf we created and specifying the output file ad.csr
```batch
certreq -new request.inf ad.csr
```

Copy the ad.csr over to your machine with openssl and create a new text file named v3ext.txt with the following contents editing the __alt_names__ to your domain:
```conf
# v3ext.txt
keyUsage=digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectKeyIdentifier=hash
subjectAltName = @alt_names
#Modify for your details. Must include the commonName in the list below also. 
#The *.example.com will allow all Domain controllers with the hostname somthing.example.com to use the cert.
[alt_names]
DNS.1 = *.example.com
DNS.2 = example.com
```

next run the following command to generate the cert for AD.
```bash
# create ad_ldaps_cert by signing the csr
# 825 days is the maximum for a cert to be trusted as dictated by the new 2019 guidelines from the CA/Browser Forum
# This is important since macOS has began to enforce this guideline
openssl x509 -req -days 825 \
    -in ad.csr \
    -CA ca.crt \
    -CAkey ca.key \
    -extfile v3ext.txt \
    -set_serial 01 \
    -out ad_ldaps_cert.crt
```

Now copy ad_ldaps_cert.crt over to the machine back to the AD Controller and accept the cert
```powershell
# accept the signed cert 
certreq -accept ad_ldaps_cert.crt
```

We can check that the cert has been imported by runnung the following powershell. We should see CN=example.com
```powershell
PS C:\LDAPS> Get-ChildItem "Cert:\LocalMachine\My"

   PSParentPath: Microsoft.PowerShell.Security\Certificate::LocalMachine\My

Thumbprint                                Subject
----------                                -------
087B0AB4E62DCE1D33323209EA81F2D58E0BF3B5  CN=example.com
```

Great now out cert is imported and ready to be used. Now we can restart the AD Controller or create the following file and run a command to tell AD to start using LDAPS

enable_ldaps.txt
```
dn:
changetype: modify
add: renewServerCertificate
renewServerCertificate: 1
-
```
Then run this command passing in the text file:
```powershell
PS C:\LDAPS> ldifde -i -f enable_ldaps.txt
Connecting to "ad01.example.com"
Logging in as current user using SSPI
Importing directory from file "enable_ldaps.txt"
Loading entries..
1 entry modified successfully.

The command has completed successfully
```

To test that we can use openssl to connect and verify we can establish a secure connection to our AD controller
```bash
openssl s_client -connect nsut-ad01.example.com:636 -CAfile ca.crt
```

### Add Cert to all domain controllers.
To add the cert and privatekey to all of our domain controllers we need to export the cert/privatekey to a pfx file to be imported on each AD DC.

First we need to get the Thumbprint of our cert to export it. Run this powershell to list your certs under the __Cert:\LocalMachine\My__ cert store. 
```powershell
PS C:\LDAPS> Get-ChildItem "Cert:\LocalMachine\My"

   PSParentPath: Microsoft.PowerShell.Security\Certificate::LocalMachine\My

Thumbprint                                Subject
----------                                -------
087B0AB4E62DCE1D33323209EA81F2D58E0BF3B5  CN=example.com
```
Specify a password and copy the thumbprint from the above output and replace it in the below command to export the cert/private key to a pfx file.
```powershell
# For security reasons we must create a password to encrypt the privatekey. Edit for YOURPASSWORD
$pfxPass = (ConvertTo-SecureString -AsPlainText -Force -String "YOURPASSWORD")
#export cert/privatekey to a pfx file.
Get-ChildItem "Cert:\LocalMachine\My\087B0AB4E62DCE1D33323209EA81F2D58E0BF3B5" | Export-PfxCertificate -FilePath LDAPS_PRIVATEKEY.pfx -Password $pfxPass
```

Now we will have a file named LDAPS_PRIVATEKEY.pfx that contains the cert and privatekey for our active directory domain controllers to use.

### Test all the Domain Controllers
The Following Powershell will test all of our Active Directory Domain Controllers for LDAPS
```powershell
##################
#### TEST ALL AD DCs for LDAPS
##################
$AllDCs = Get-ADDomainController -Filter * -Server nsuok.edu | Select-Object Hostname
 foreach ($dc in $AllDCs) {
	$LDAPS = [ADSI]"LDAP://$($dc.hostname):636"
	#write-host $LDAPS
	try {
   	$Connection = [adsi]($LDAPS)
	} Catch {
	}
	If ($Connection.Path) {
   	Write-Host "Active Directory server correctly configured for SSL, test connection to $($LDAPS.Path) completed."
	} Else {
   	Write-Host "Active Directory server not configured for SSL, test connection to LDAP://$($dc.hostname):636 did not work."
	}
 }
```

### Congratulations
You now have all your domain controllers configured to use Secure LDAPS. But this is just half the battle we now need to configure all of our Services, Apps, AD joined macOS computers and Servers to use LDAPS.

### How to find what systems (Servers) are using insecure LDAP Binds
Read my next article to learn how to turn on logging in Active Directory and export the logs to CSV using powershell.
Coming soon.

