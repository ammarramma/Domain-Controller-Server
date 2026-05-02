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

# Remote Desktop
```Approved
```

# ADFS

# DFS
