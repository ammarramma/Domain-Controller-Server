# DHCP
```Powershell
Install-WindowsFeature DHCP -includeManagementTools

Add-DHCPServerInDC -DNSName "a3n4.com" -IpAddress 192.168.56.10

Add-DhcpServerV4Scope -Name "Scope-a3n4" -StartRange 192.168.56.11 -EndRange 192.168.56.20 -SubnetMask 255.255.255.0 -State Actives

Set-DHCPServerV4OptionValue -ScopeId 192.168.56.0 -Router 192.168.56.1

Set-DHCPServerV4OptionValue -ScopeId 192.168.56.0 -DnsServer 192.168.56.10 -DnsDomain "a3n4.com"

Set-DHCPServerV4Scope -ScopeId 192.168.56.0 -LeaseDuration ([TimeSpan]::FromDays(365))
```

# DNS
```Powershell
Install-WindowsFeature -Name DNS -IncludeManagementTools

Add-DnsServerPrimaryZone -NetworkId "192.168.56.0/24" -ZoneFile "56.168.192.in-addr.arpa.dns"

Add-DnsServerResourceRecordPtr -Name "10" -ZoneName "56.168.192.in-addr.arpa" -PtrDomainName "www.a3n4.com"

Add-DnsServerResourceRecordA -Name "www" -ZoneName "a3n4.com" -Ipv4Address "192.168.56.10"

```

# Join Domain
```Approved
```

# User & Group
```powershell
New-AdOrganizationalUnit -Name "a3n4_Staff" -Path "DC=a3n4, DC=com"

New-AdGroup -Name "HR_Staff" -GroupScope Global -Path "OU=a3n4_Staff, DC=a3n4, DC=com" -GroupCategory Security

New-ADUser -Name "Ammar" -GivenName "Ammar" -Surname "" -SamAccountName "Ammar" -UserPrincipalName "Ammar@a3n4.com" -Path "OU=a3n4_Staff,DC=a3n4,DC=com" -AccountPassword (ConvertTo-SecureString "##ammarTi2b" -AsPlainText -Force) -Enabled $true

Enable-ADAccount -Identity "Ammar" 
Get-ADUser -Filter * -SearchBase "OU=a3n4_Staff,DC=a3n4,DC=com" | Enable-ADAccount

Add-AdGroupMember -Identity "Nama_Group" -Members "Nama_User"
```

# IIS
```powershell
Install-WindowsFeature -Name Web-Server -IncludeManagementTools

New-Item -Path "C:\inetpub\a3n4web" -ItemType directory

New-Website -Name "a3n4web" -Port 80 -IpAddress "*" -HostHeader "a3n4.com" -PhysicalPath "C:\inetpub\a3n4web"

Copy-Item -Path "z:\website_iis\*" -Destination "C:\inetpub\a3n4web" -Recurse

Install-WindowsFeature Web-Static-Content

New-SelfSignedCertificate -DnsName "a3n4.com" -CertStoreLocation "cert:\LocalMachine\My"

Import-Module WebAdministration

New-WebBinding -Name "a3n4web" -Protocol https -Port 443 -IpAddress "*" -HostHeader "a3n4.com"

Get-Item "cert:\LocalMachine\My\CEEF3D52BC7E3A5E4DF6B30524EC5D3E9B0CC050" | New-Item "IIS:SslBindings\0.0.0.0!443!a3n4.com"
```
# IIS Two
```powershell
Install-WindowsFeature -Name Web-Server -IncludeManagementTools
```

# Remote Desktop
```Approved
```

# ADFS
```Powershell

```

# DFS
## Steps Install
```Powershell
Start-Service RemoteRegistry
Set-Service RemoteRegistry -StartupType Automatic

Install-WindowsFeature Fs-Dfs-NameSpace -IncludeManagementTools

# DFS Root (Namespace storage)
New-Item -Path "C:\DFSRoots\a3n4_shared" -ItemType Directory
New-SmbShare -Name "DFSa3n4" -Path "C:\DFSRoots\a3n4_shared" -FullAccess "a3n4\Administrator"

New-DfsnRoot -Type DomainV2 -Path "\\a3n4.com\data_a3n4" -TargetPath "\\SRV01\DFSa3n4"

# Data Share
New-Item -ItemType Directory -Path "C:\Shared"
New-SmbShare -Name "data_share" -Path "C:\Shared" -FullAccess "a3n4\Administrator"

# Folder struktur
New-Item -ItemType Directory -Path "C:\Shared\AllStaff"
New-Item -ItemType Directory -Path "C:\Shared\HRStaff"
New-Item -ItemType Directory -Path "C:\Shared\MarketingStaff"
New-Item -ItemType Directory -Path "C:\Shared\FinanceStaff"

# NTFS Permission (Data)
Icacls "C:\Shared\AllStaff" /inheritance:r
Icacls "C:\Shared\AllStaff" /grant "a3n4\All_Staff:(OI)(CI)M" "a3n4\Administrator:(OI)(CI)F" "SYSTEM:(OI)(CI)F"

Icacls "C:\Shared\HRStaff" /inheritance:r
Icacls "C:\Shared\HRStaff" /grant "a3n4\HR_Staff:(OI)(CI)M" "a3n4\Administrator:(OI)(CI)F" "SYSTEM:(OI)(CI)F"

Icacls "C:\Shared\FinanceStaff" /inheritance:r
Icacls "C:\Shared\FinanceStaff" /grant "a3n4\Finance_Staff:(OI)(CI)M" "a3n4\Administrator:(OI)(CI)F" "SYSTEM:(OI)(CI)F"

Icacls "C:\Shared\MarketingStaff" /inheritance:r
Icacls "C:\Shared\MarketingStaff" /grant "a3n4\Marketing_Staff:(OI)(CI)M" "a3n4\Administrator:(OI)(CI)F" "SYSTEM:(OI)(CI)F"

# DFS Folder Mapping
New-DfsnFolder -Path "\\a3n4.com\data_a3n4\AllStaff" -TargetPath "\\SRV01\data_share\AllStaff"
New-DfsnFolder -Path "\\a3n4.com\data_a3n4\HRStaff" -TargetPath "\\SRV01\data_share\HRStaff"
New-DfsnFolder -Path "\\a3n4.com\data_a3n4\MarketingStaff" -TargetPath "\\SRV01\data_share\MarketingStaff"
New-DfsnFolder -Path "\\a3n4.com\data_a3n4\FinanceStaff" -TargetPath "\\SRV01\data_share\FinanceStaff"

# Share Permission
Grant-SmbShareAccess -Name DFSa3n4 -Account "a3n4\All_Staff" -AccessRight Read -Force
Grant-SmbShareAccess -Name DFSa3n4 -Account "a3n4\HR_Staff" -AccessRight Read -Force
Grant-SmbShareAccess -Name DFSa3n4 -Account "a3n4\Finance_Staff" -AccessRight Read -Force
Grant-SmbShareAccess -Name DFSa3n4 -Account "a3n4\Marketing_Staff" -AccessRight Read -Force

Grant-SmbShareAccess -Name data_share -Account "a3n4\All_Staff" -AccessRight Change -Force
Grant-SmbShareAccess -Name data_share -Account "a3n4\HR_Staff" -AccessRight Change -Force
Grant-SmbShareAccess -Name data_share -Account "a3n4\Finance_Staff" -AccessRight Change -Force
Grant-SmbShareAccess -Name data_share -Account "a3n4\Marketing_Staff" -AccessRight Change -Force

# Enable ABE
Set-DfsnRoot -Path "\\a3n4.com\data_a3n4" -EnableAccessBasedEnumeration $true

# Plus
takeown /f "C:\DFSRoots"

Icacls "C:\DFSRoots" /setowner "NT AUTHORITY\SYSTEM" /T /C

Icacls "C:\DFSRoots"

takeown /f "C:\DFSRoots\a3n4_shared\AllStaff" /r /d y
Icacls "C:\DFSRoots\a3n4_shared\AllStaff" /inheritance:r
Icacls "C:\DFSRoots\a3n4_shared\AllStaff" /grant "a3n4\All_Staff:(OI)(CI)M"
Icacls "C:\DFSRoots\a3n4_shared\AllStaff"

takeown /f "C:\DFSRoots\a3n4_shared\HRStaff" /r /d y
Icacls "C:\DFSRoots\a3n4_shared\HRStaff" /inheritance:r
Icacls "C:\DFSRoots\a3n4_shared\HRStaff" /grant "a3n4\HR_Staff:(OI)(CI)M"
Icacls "C:\DFSRoots\a3n4_shared\HRStaff"

takeown /f "C:\DFSRoots\a3n4_shared\MarketingStaff" /r /d y
Icacls "C:\DFSRoots\a3n4_shared\MarketingStaff" /grant "a3n4\Marketing_Staff:(OI)(CI)M"
Icacls "C:\DFSRoots\a3n4_shared\MarketingStaff"

takeown /f "C:\DFSRoots\a3n4_shared\FinanceStaff" /r /d y
Icacls "C:\DFSRoots\a3n4_shared\FinanceStaff" /grant "a3n4\Finance_Staff:(OI)(CI)M"
Icacls "C:\DFSRoots\a3n4_shared\FinanceStaff"
```
## Remove Step
```Powershell
# Hapus DFS Folder
Remove-DfsnFolder -Path "\\a3n4.com\data_a3n4\AllStaff" -Force
Remove-DfsnFolder -Path "\\a3n4.com\data_a3n4\HRStaff" -Force
Remove-DfsnFolder -Path "\\a3n4.com\data_a3n4\MarketingStaff" -Force
Remove-DfsnFolder -Path "\\a3n4.com\data_a3n4\FinanceStaff" -Force

# Hapus DFS Namespace
Remove-DfsnRoot -Path "\\a3n4.com\data_a3n4" -Force

# Hapus DFS Folder
Remove-DfsnFolder -Path "\\a3n4.com\data_a3n4\AllStaff" -Force
Remove-DfsnFolder -Path "\\a3n4.com\data_a3n4\HRStaff" -Force
Remove-DfsnFolder -Path "\\a3n4.com\data_a3n4\MarketingStaff" -Force
Remove-DfsnFolder -Path "\\a3n4.com\data_a3n4\FinanceStaff" -Force
# OR
Remove-DfsnFolder -Path "\\a3n4.com\data_a3n4\*" -Force

# Hapus Share
Remove-SmbShare -Name "data_share" -Force
Remove-SmbShare -Name "DFSa3n4" -Force

# Hapus Folder
Remove-Item "C:\Shared" -Recurse -Force
Remove-Item "C:\DFSRoots" -Recurse -Force

# (Optional) Uninstall feature
Uninstall-WindowsFeature Fs-Dfs-NameSpace
```
