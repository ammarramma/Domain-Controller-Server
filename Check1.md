# 🧠 Windows Server Infrastructure Guide — A3N4

Dokumen ini bukan sekadar kumpulan command. Ini peta berpikir.  
Kalau kamu asal copy-paste tanpa ngerti, ya selamat... kamu cuma jadi robot yang lebih murah dari aku.

---

# 📌 1. SCONFIG (IP SET)
## 🎯 Tujuan
Menentukan identitas dasar server (IP & hostname).

## 🧠 Konsep
Server itu pusat. Kalau IP berubah-ubah, semua client bakal “lost”.  
Static IP = alamat rumah tetap.

## ⚙️ Command
```cmd
SConfig
```

Auto-launch saat login:
```powershell
set-sconfig -autoLaunch $true
```

## 🪜 Langkah
1. Ubah hostname → opsi `2`
2. Set network → opsi `3`
   - Pilih interface
   - Set IP → pilih `1`
   - Gunakan static (`S`)
   - IP: `192.168.56.10`
   - Subnet: Enter (default)
   - Gateway: kosong
3. Restart → opsi `13`

---

# 🏛️ 2. AD DS (ACTIVE DIRECTORY)
## 🎯 Tujuan
Membuat sistem identitas terpusat (user & komputer).

## 🧠 Konsep
Tanpa AD → semua komputer itu random stranger.  
Dengan AD → semua jadi warga negara.

## ⚙️ Install
```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools

Install-ADDSForest `
-DomainName "A3N4.com" `
-DomainNetBiosName "A3N4" `
-SafeModeAdministratorPassword (ConvertTo-SecureString "Aclsdmin123" -AsPlainText -Force)
```

## 🔍 Verifikasi
```powershell
Get-WindowsFeature AD-Domain-Services
Get-Service NTDS
$env:USERDOMAIN
```

## ❌ Remove
```powershell
Uninstall-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

---

# 📡 3. DHCP
## 🎯 Tujuan
Memberikan IP otomatis ke client.

## 🧠 Konsep
Admin malas? DHCP solusinya.  
Manual IP itu cocok kalau kamu punya 3 device, bukan 300.

## ⚙️ Install & Config
```powershell
Install-WindowsFeature DHCP -IncludeManagementTools

Add-DhcpServerInDC -DNSName "a3n4.com" -IpAddress 192.168.56.10

Add-DhcpServerV4Scope `
-Name "Scope-a3n4" `
-StartRange 192.168.56.11 `
-EndRange 192.168.56.20 `
-SubnetMask 255.255.255.0 `
-State Active

Set-DhcpServerV4OptionValue -ScopeId 192.168.56.0 -Router 192.168.56.1
Set-DhcpServerV4OptionValue -ScopeId 192.168.56.0 -DnsServer 192.168.56.10 -DnsDomain "a3n4.com"

Set-DhcpServerV4Scope -ScopeId 192.168.56.0 -LeaseDuration ([TimeSpan]::FromDays(365))
```

## 🔍 Verifikasi
```powershell
Get-WindowsFeature DHCP
Get-Service DHCPServer
Get-DhcpServerV4OptionValue -ScopeId 192.168.56.0
Get-DhcpServerInDC
Get-DhcpServerV4ScopeStatistics -ScopeId 192.168.56.0
Get-DhcpServerV4Lease -ScopeId 192.168.56.0
```

## ❌ Remove
```powershell
Remove-DhcpServerV4Scope -ScopeId 192.168.56.0 -Force
Remove-DhcpServerInDC -DNSName "a3n4.com" -IpAddress 192.168.56.10
Uninstall-WindowsFeature DHCP -IncludeManagementTools
Restart-Computer
```

---

# 🌐 4. DNS
## 🎯 Tujuan
Mapping nama ke IP.

## 🧠 Konsep
Manusia ingat nama, bukan angka.  
DNS = penerjemah.

## ⚙️ Install
```powershell
Install-WindowsFeature DNS -IncludeManagementTools
```

## 🔁 Reverse Zone
```powershell
Add-DnsServerPrimaryZone `
-NetworkId "192.168.56.0/24" `
-ZoneFile "56.168.192.in-addr.arpa.dns"
```

## 🔗 PTR Record
```powershell
Add-DnsServerResourceRecordPtr `
-Name "10" `
-ZoneName "56.168.192.in-addr.arpa" `
-PtrDomainName "www.a3n4.com"
```

## 🌍 A Record
```powershell
Add-DnsServerResourceRecordA `
-Name "www" `
-ZoneName "a3n4.com" `
-IPv4Address "192.168.56.10"
```

## 🔍 Verifikasi
```powershell
Get-DnsServerZone
Get-DnsServerResourceRecord -ZoneName "a3n4.com"

nslookup www.a3n4.com
ping www.a3n4.com
```

## ❌ Remove
```powershell
Remove-DnsServerResourceRecord -Name "www" -ZoneName "a3n4.com" -RRType A -Force
Remove-DnsServerZone -Name "56.168.192.in-addr.arpa" -Force
Remove-WindowsFeature DNS -IncludeManagementTools
```

---

# 🧾 5. JOIN DOMAIN
## 🎯 Tujuan
Client masuk ke domain.

## 🧠 Konsep
Tanpa ini, client cuma numpang lewat.  
Dengan ini, dia resmi jadi warga.

## 🪜 Langkah
```
sysdm.cpl → Change → Domain → a3n4.com
```

## 🔍 Verifikasi
```cmd
ipconfig
```

---

# 👥 6. USER, GROUP, OU
## 🎯 Tujuan
Manajemen identitas & akses.

## ⚙️ OU
```powershell
New-ADOrganizationalUnit -Name "a3n4_Staff" -Path "DC=a3n4,DC=com"
```

## ⚙️ Group
```powershell
New-ADGroup -Name "HR_Staff" -GroupScope Global -GroupCategory Security -Path "OU=a3n4_Staff,DC=a3n4,DC=com"
```

## ⚙️ User
```powershell
New-ADUser `
-Name "Ammar" `
-SamAccountName "Ammar" `
-UserPrincipalName "Ammar@a3n4.com" `
-Path "OU=a3n4_Staff,DC=a3n4,DC=com" `
-AccountPassword (ConvertTo-SecureString "##ammarTi2b" -AsPlainText -Force) `
-Enabled $true
```

## 🔗 Membership
```powershell
Add-ADGroupMember -Identity "HR_Staff" -Members "Ammar"
```

## 🔍 Verifikasi
```powershell
Get-ADUser -Filter *
Get-ADGroupMember -Identity "HR_Staff"
```

---

# 🌍 7. IIS (WEB SERVER)
## 🎯 Tujuan
Menjalankan website.

## ⚙️ Install
```powershell
Install-WindowsFeature Web-Server -IncludeManagementTools
```

## ⚙️ Setup
```powershell
New-Item -Path "C:\inetpub\a3n4web" -ItemType Directory

New-Website `
-Name "a3n4web" `
-Port 80 `
-HostHeader "a3n4.com" `
-PhysicalPath "C:\inetpub\a3n4web"
```

## 🔐 HTTPS
```powershell
New-SelfSignedCertificate -DnsName "a3n4.com" -CertStoreLocation "cert:\LocalMachine\My"
```

---

# 🗂️ 8. DFS
## 🎯 Tujuan
Centralized file sharing.

## 🧠 Konsep
User lihat satu folder, padahal backend bisa banyak server.

## ⚙️ Setup Singkat
```powershell
Install-WindowsFeature Fs-Dfs-NameSpace -IncludeManagementTools

New-SmbShare -Name "DFSa3n4" -Path "C:\DFSRoots\a3n4_shared"

New-DfsnRoot -Type DomainV2 -Path "\\a3n4.com\data_a3n4" -TargetPath "\\SRV01\DFSa3n4"
```

---

# 📦 9. FSRM
## 🎯 Tujuan
Limit storage user.

## ⚙️ Setup
```powershell
Install-WindowsFeature FS-Resource-Manager -IncludeManagementTools

New-FsrmQuota -Path "C:\Shared\AllStaff" -Template "250MB"
```

---

# 🧭 10. REMOTE DESKTOP
## 🎯 Tujuan
Akses server jarak jauh.

## 🪜 Langkah
```
SConfig → 7 → Enable
```

---

# 🧩 11. ADFS
## 🎯 Tujuan
Single Sign-On berbasis web.

## ⚙️ Setup
```powershell
Install-WindowsFeature ADFS-Federation -IncludeManagementTools

New-SelfSignedCertificate -DnsName "adfs.a3n4.com" -CertStoreLocation "cert:\LocalMachine\My"

Install-ADFSFarm `
-CertificateThumbprint "<thumbprint>" `
-FederationServiceName "adfs.a3n4.com"
```

---

# 📡 12. FTP
## 🎯 Tujuan
Transfer file via jaringan.

## ⚙️ Setup
```powershell
Install-WindowsFeature Web-FTP-Server

New-WebFtpSite -Name "A3N4FTP" -PhysicalPath "C:\FTPRoot" -Port 21
```

---

# 📧 13. MAIL SERVER
## 🎯 Tujuan
Email internal domain.

## ⚙️ DNS
```powershell
Add-DnsServerResourceRecordA -Name "mail" -ZoneName "a3n4.com" -IPv4Address "192.168.56.10"
```

---

# 🔐 14. VPN
## 🎯 Tujuan
Remote access aman.

## ⚙️ Setup
```powershell
Install-WindowsFeature RemoteAccess -IncludeManagementTools

Install-RemoteAccess -VpnType VPN
```

---

# ⏱️ 15. NTP
## 🎯 Tujuan
Sinkronisasi waktu.

## ⚙️ Setup
```powershell
w32tm /config /manualpeerlist:"time.windows.com" /syncfromflags:manual /update
w32tm /resync
```

---

# 🧩 PENUTUP

Kalau ini terasa “rapi”, itu karena kamu akhirnya berhenti bikin dokumentasi kayak catatan warung.

Struktur ini:
- Pisahin konsep vs command
- Minimalis tapi jelas
- Bisa dibaca manusia, bukan cuma kamu pas panik jam 2 pagi

Masalah klasik orang bikin doc:
nulis semuanya, tapi ga mikirin yang baca.

Sekarang? Lumayan. Udah bisa dibilang “niat”.