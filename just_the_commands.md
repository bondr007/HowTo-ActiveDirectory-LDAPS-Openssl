# Get going quick and skip the reading by using this guide


## prereqs
machine with openssl
AD Controller with Windows Server 2012 or greater.


## Warning
This guide does not explain any of the commands. You might want to read the full README.MD first then use this to quickly go through the commands.
It is also assumed you are using a Linux machine for openssl


## env setup
Mount your AD controller c:\ drive over SMB. 
In Ubuntu using the Nautilus (File Browser) you can mount the drive by clicking on the __+ Other Locations__ button in the lower left of the window. 
Then in the textbox enter the hostname of your ad controller like the following:
smb://ad01.example.com/c$

This will prompt you for your Active Directory Credentials 

Once mounted right click in some empty space in the windows and select __Open in Terminal__

Now we can make a folder named LDAPS and download the contents of this repo into it.

## Edit the files
We need to edit the following files replacing __example.com__ with our domain
* ca_san.conf
* request.inf
* v3ext.conf

## The Commands
We will be switching between our linux terminal and a Powershell window on a domain controller for these commands.
I will denote each by specifying either __linux__ or __ADDC__ before each line.

__linux__
```bash
# generate the ca key, create a password and keep it for use throught this guide.
openssl genrsa -des3 -out ca.key 4096

```
__linux__
```bash
# create ca cert with valid of 10 years with info based off the provided ca_san.conf file, it will prompt for the password we created earlier 
openssl req -new -x509 \
    -extensions v3_ca \
    -days 3650 \
    -key ca.key \
    -out ca.crt \
    -config ca_san.conf
```
__ADDC__
```powershell
#import the cert as a trusted CA on the domain controller
Import-Certificate -FilePath ca.crt  -CertStoreLocation 'Cert:\LocalMachine\Root' 
```
__ADDC__
```powershell
#create the csr on the domain controller using the values in request.inf file
certreq -new request.inf ad.csr
```
__linux__
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
__ADDC__
```powershell
# accept the signed cert 
certreq -accept ad_ldaps_cert.crt
```
__ADDC__
```powershell
# tell active directory to use the cert we created
ldifde -i -f enable_ldaps.txt
```

### We now have this AD controller configured to use LDAPS. 
To export the cert and import it on the rest of domain controllers run the following powershell


__ADDC__
```powershell
#Get the thumbprint of our cert and replace the value in the next command
Get-ChildItem "Cert:\LocalMachine\My"
```

__ADDC__
```powershell
$pfxPass = (ConvertTo-SecureString -AsPlainText -Force -String "YOURPASSWORD")
#export cert/privatekey to a pfx file
Get-ChildItem "Cert:\LocalMachine\My\ASDF_YOUR_THUMBPRINT_HERE" | Export-PfxCertificate -FilePath LDAPS_PRIVATEKEY.pfx -Password $pfxPass
```
Now we will have a file named LDAPS_PRIVATEKEY.pfx that contains the cert and privatekey for our active directory domain controllers to use.

```powershell
$AllDCs = Get-ADDomainController -Filter * -Server nsuok.edu | Select-Object Hostname

Invoke-Command -ComputerName $AllDCs.Hostname -ScriptBlock {
    #Edit this path to a share where you have the ca.crt file stored
    $caPath="\\ad01.example.com\c$\LDAPS\ca.crt"
    #Edit this path to a share where you have the pfx file stored
    $pfxPath="\\ad01.example.com\c$\LDAPS\LDAPS_PRIVATEKEY.pfx"
    #Edit for yourpassword
    $pfxPass = (ConvertTo-SecureString -AsPlainText -Force -String "YOURPASSWORD")

    Import-Certificate -FilePath $caPath -CertStoreLocation 'Cert:\LocalMachine\Root' -Verbose
    Import-PfxCertificate -FilePath $pfxPath -CertStoreLocation Cert:\LocalMachine\My -Password $pfxPass

    #edit path to enable_ldaps.txt
    ldifde -i -f \\ad01.example.com\c$\LDAPS\enable_ldaps.txt
}
```

