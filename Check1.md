# WINDOWS SERVER INFRASTRUCTURE GUIDE - A3N4

Dokumen ini sudah dirapikan dengan prinsip:
- urutan logis (dari dasar → service)
- command lengkap (tidak dipotong diam-diam)
- penjelasan singkat tapi jelas (tidak ceramah kosong)

Kalau masih gagal, itu bukan salah dokumentasi lagi.

====================================================================

SECTION: SCONFIG (IP SET)

TUJUAN:
Menentukan identitas dasar server (hostname + static IP)

COMMAND:
cmd:
SConfig

powershell:
set-sconfig -autoLaunch $true

LANGKAH:
- pilih 2 → ubah hostname
- pilih 3 → network
- pilih interface
- pilih 1 → set IP
- pilih S (static)
- isi:
  IP: 192.168.56.10
  subnet: enter (default 255.255.255.0)
  gateway: kosong
- pilih 13 → restart

====================================================================

SECTION: AD DS

TUJUAN:
Membuat domain controller dan sistem identitas

INSTALL:
powershell:
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

Install-ADDSForest `
-DomainName "A3N4.com" `
-DomainNetBiosName "A3N4" `
-SafeModeAdministratorPassword (ConvertTo-SecureString "Aclsdmin123" -AsPlainText -Force)

VERIFIKASI:
Get-WindowsFeature AD-Domain-Services
Get-Service NTDS
$env:USERDOMAIN

REMOVE:
Uninstall-WindowsFeature AD-Domain-Services -IncludeManagementTools

====================================================================

SECTION: DHCP

INSTALL:
Install-WindowsFeature DHCP -IncludeManagementTools

Add-DhcpServerInDC -DNSName "a3n4.com" -IpAddress 192.168.56.10

Add-DhcpServerV4Scope `
-Name "Scope-a3n4" `
-StartRange 192.168.56.11 `
-EndRange 192.168.56.20 `
-SubnetMask 255.255.255.0 `
-State Active

Set-DhcpServerV4OptionValue -ScopeId 192.168.56.0 -Router 192.168.56.1

Set-DhcpServerV4OptionValue -ScopeId 192.168.56.0 `
-DnsServer 192.168.56.10 `
-DnsDomain "a3n4.com"

Set-DhcpServerV4Scope `
-ScopeId 192.168.56.0 `
-LeaseDuration ([TimeSpan]::FromDays(365))

VERIFIKASI:
Get-WindowsFeature DHCP
Get-Service DHCPServer
Get-DhcpServerV4OptionValue -ScopeId 192.168.56.0
Get-DhcpServerInDC
Get-DhcpServerV4ScopeStatistics -ScopeId 192.168.56.0
Get-DhcpServerV4Lease -ScopeId 192.168.56.0

REMOVE:
Remove-DhcpServerV4Scope -ScopeId 192.168.56.0 -Force
Remove-DhcpServerInDC -DNSName "a3n4.com" -IpAddress 192.168.56.10
Uninstall-WindowsFeature DHCP -IncludeManagementTools
Restart-Computer

====================================================================

SECTION: DNS

INSTALL:
Install-WindowsFeature -Name DNS -IncludeManagementTools

REVERSE ZONE:
Add-DnsServerPrimaryZone `
-NetworkId "192.168.56.0/24" `
-ZoneFile "56.168.192.in-addr.arpa.dns"

PTR:
Add-DnsServerResourceRecordPtr `
-Name "10" `
-ZoneName "56.168.192.in-addr.arpa" `
-PtrDomainName "www.a3n4.com"

A RECORD:
DnsCmd /recordAdd a3n4.com www A 192.168.56.10

Add-DnsServerResourceRecordA `
-Name "www" `
-ZoneName "a3n4.com" `
-IPv4Address "192.168.56.10"

VERIFIKASI:
Get-DnsServerZone
Get-DnsServerResourceRecord -ZoneName "a3n4.com"
nslookup www.a3n4.com
ping www.a3n4.com

REMOVE:
Remove-DnsServerResourceRecord -Name "www" -ZoneName "a3n4.com" -RRType A -Force
Remove-DnsServerResourceRecord -Name "10" -ZoneName "56.168.192.in-addr.arpa" -RRType PTR -Force
Remove-DnsServerZone -Name "56.168.192.in-addr.arpa" -Force
Remove-WindowsFeature -Name DNS -IncludeManagementTools

====================================================================

SECTION: JOIN DOMAIN

LANGKAH:
run:
sysdm.cpl

- change
- pilih domain
- isi: a3n4.com
- login credential
- restart

VERIFIKASI:
ipconfig
ping a3n4.com

====================================================================

SECTION: USER, GROUP, OU

OU:
New-ADOrganizationalUnit -Name "a3n4_Staff" -Path "DC=a3n4,DC=com"

GROUP:
New-ADGroup -Name "All_Staff" -GroupScope Global -GroupCategory Security -Path "OU=a3n4_Staff,DC=a3n4,DC=com"
New-ADGroup -Name "Finance_Staff" -GroupScope Global -GroupCategory Security -Path "OU=a3n4_Staff,DC=a3n4,DC=com"
New-ADGroup -Name "HR_Staff" -GroupScope Global -GroupCategory Security -Path "OU=a3n4_Staff,DC=a3n4,DC=com"
New-ADGroup -Name "Marketing_Staff" -GroupScope Global -GroupCategory Security -Path "OU=a3n4_Staff,DC=a3n4,DC=com"

USER:
New-ADUser -Name "Ammar" -GivenName "Ammar" -SamAccountName "Ammar" -UserPrincipalName "Ammar@a3n4.com" -Path "OU=a3n4_Staff,DC=a3n4,DC=com" -AccountPassword (ConvertTo-SecureString "##ammarTi2b" -AsPlainText -Force) -Enabled $true

MEMBER:
Add-ADGroupMember -Identity "All_Staff" -Members "Ammar"

VERIFIKASI:
Get-ADUser -Filter *
Get-ADGroup -Filter *
Get-ADGroupMember -Identity "All_Staff"

====================================================================

SECTION: IIS

Install-WindowsFeature -Name Web-Server -IncludeManagementTools

New-Item -Path "C:\inetpub\a3n4web" -ItemType Directory

New-Website `
-Name "a3n4web" `
-Port 80 `
-IpAddress "*" `
-HostHeader "a3n4.com" `
-PhysicalPath "C:\inetpub\a3n4web"

Copy-Item -Path "Z:\web_iis\*" -Destination "C:\inetpub\a3n4web"

Install-WindowsFeature Web-Static-Content

New-SelfSignedCertificate -DnsName "a3n4.com" -CertStoreLocation "cert:\LocalMachine\My"

Import-Module WebAdministration

New-WebBinding -Name "a3n4web" -Protocol https -Port 443 -IpAddress "*" -HostHeader "a3n4.com"

Get-Item "cert:\LocalMachine\My\<ThumbPrint>" | New-Item "IIS:SslBindings\0.0.0.0!443!a3n4.com"

====================================================================

SECTION: DFS

Start-Service RemoteRegistry
Set-Service RemoteRegistry -StartupType Automatic

Install-WindowsFeature Fs-Dfs-NameSpace -IncludeManagementTools

New-Item -Path "C:\DFSRoots\a3n4_shared" -ItemType Directory

New-SmbShare -Name "DFSa3n4" -Path "C:\DFSRoots\a3n4_shared" -FullAccess "a3n4\Administrator"

New-DfsnRoot -Type DomainV2 -Path "\\a3n4.com\data_a3n4" -TargetPath "\\SRV01\DFSa3n4"

New-Item -ItemType Directory -Path "C:\Shared"

New-SmbShare -Name "data_share" -Path "C:\Shared" -FullAccess "a3n4\Administrator"

New-Item -ItemType Directory -Path "C:\Shared\AllStaff"
New-Item -ItemType Directory -Path "C:\Shared\HRStaff"
New-Item -ItemType Directory -Path "C:\Shared\MarketingStaff"
New-Item -ItemType Directory -Path "C:\Shared\FinanceStaff"

====================================================================

SECTION: FSRM

Install-WindowsFeature -Name FS-Resource-Manager -IncludeManagementTools

Restart-Service SrmSvc

New-FsrmQuotaTemplate -Name "250MB" -Description "Maksimum 250MB" -Size 250MB

New-FsrmQuota -Path "C:\Shared\AllStaff" -Template "250MB"

====================================================================

SECTION: REMOTE DESKTOP

SConfig → 7 → Enable → 1 → restart

====================================================================

SECTION: ADFS

Install-WindowsFeature ADFS-Federation -IncludeManagementTools

Add-DnsServerResourceRecordA -Name "adfs" -ZoneName "a3n4.com" -Ipv4Address 192.168.56.10

New-SelfSignedCertificate -DnsName "adfs.a3n4.com" -CertStoreLocation "cert:\LocalMachine\My"

Add-KdsRootKey -EffectiveTime ((Get-Date).AddDays(1))

====================================================================

SECTION: FTP

Install-WindowsFeature -Name Web-FTP-Server, Web-FTP-Service, Web-Mgmt-Service, Web-Server

New-Item -Path "C:\FTPRoot" -ItemType Directory

Import-Module WebAdministration

New-WebFtpSite -Name "A3N4FTP" -PhysicalPath "C:\FTPRoot" -Port 21 -IPAddress "*"

====================================================================

SECTION: VPN

Install-WindowsFeature -Name RemoteAccess -IncludeManagementTools

Install-RemoteAccess -VpnType VPN

====================================================================

SECTION: NTP

w32tm /Config /ManualPeerList:"time.windows.com" /SyncFromFlags:Manual /Update
w32tm /Resync

====================================================================

SECTION: MAIL (DNS ONLY CORE)

Add-DnsServerResourceRecordA -Name "mail" -ZoneName "a3n4.com" -IPv4Address "192.168.56.10"

Add-DnsServerResourceRecordCName -Name "autodiscover" -HostNameAlias "mail.a3n4.com" -ZoneName "a3n4.com"

Add-DnsServerResourceRecord -ZoneName "a3n4.com" -Name "@" -Txt -DescriptiveText "v=spf1 ip4:192.168.56.10 -all"

====================================================================

END OF DOCUMENT