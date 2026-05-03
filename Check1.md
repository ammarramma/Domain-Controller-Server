# Windows Server Infrastructure Guide — A3N4 (Explained Edition)

Dokumen ini versi “manusia mikir”.  
Setiap bagian bukan cuma command, tapi dijelaskan kenapa dan apa efeknya.  
Kalau masih copy-paste tanpa ngerti, itu bukan salah dokumentasi.

---

## 1. SCONFIG (IP SET)
### Fungsi
Menentukan identitas dasar server: hostname dan IP statis.

### Kenapa penting
Server harus bisa ditemukan secara konsisten. DHCP di server itu ide buruk kecuali kamu suka debugging tanpa arah.

### Command
```cmd
SConfig
```
Penjelasan:
Membuka menu konfigurasi cepat berbasis CLI untuk Windows Server Core.

```powershell
set-sconfig -autoLaunch $true
```
Penjelasan:
Mengatur agar SConfig otomatis muncul setiap login. Berguna kalau kamu sering konfigurasi server minimal.

### Langkah
- Opsi 2 → ubah hostname → memberi identitas unik server
- Opsi 3 → network:
  - pilih interface
  - set IP static:
    IP: 192.168.56.10 → alamat tetap server
    Subnet: default → menentukan jaringan lokal
    Gateway: kosong → karena tidak keluar jaringan
- Opsi 13 → restart agar perubahan diterapkan

---

## 2. AD DS (ACTIVE DIRECTORY)
### Fungsi
Membuat sistem manajemen identitas terpusat.

### Kenapa penting
Tanpa AD, setiap komputer hidup sendiri. Dengan AD, semua terkontrol.

### Command
```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```
Penjelasan:
Menginstall role Active Directory beserta tools manajemennya.

```powershell
Install-ADDSForest `
-DomainName "A3N4.com" `
-DomainNetBiosName "A3N4" `
-SafeModeAdministratorPassword (ConvertTo-SecureString "Aclsdmin123" -AsPlainText -Force)
```
Penjelasan:
- DomainName → nama domain utama
- NetBIOS → nama pendek domain
- SafeModePassword → password recovery AD

### Verifikasi
```powershell
Get-WindowsFeature AD-Domain-Services
```
Penjelasan:
Memastikan role sudah terinstall.

```powershell
Get-Service NTDS
```
Penjelasan:
Service utama AD, harus running.

```powershell
$env:USERDOMAIN
```
Penjelasan:
Menunjukkan domain aktif.

---

## 3. DHCP
### Fungsi
Memberikan IP otomatis ke client.

### Kenapa penting
Manual IP itu buang waktu dan rawan konflik.

### Command
```powershell
Install-WindowsFeature DHCP -IncludeManagementTools
```
Penjelasan:
Menginstall service DHCP.

```powershell
Add-DhcpServerInDC -DNSName "a3n4.com" -IpAddress 192.168.56.10
```
Penjelasan:
Mendaftarkan DHCP ke Active Directory agar diizinkan berjalan.

```powershell
Add-DhcpServerV4Scope `
-Name "Scope-a3n4" `
-StartRange 192.168.56.11 `
-EndRange 192.168.56.20 `
-SubnetMask 255.255.255.0 `
-State Active
```
Penjelasan:
- Range → IP yang bisa dipinjam client
- Subnet → batas jaringan
- Active → langsung digunakan

```powershell
Set-DhcpServerV4OptionValue -ScopeId 192.168.56.0 -Router 192.168.56.1
```
Penjelasan:
Menentukan gateway untuk client.

```powershell
Set-DhcpServerV4OptionValue -ScopeId 192.168.56.0 -DnsServer 192.168.56.10 -DnsDomain "a3n4.com"
```
Penjelasan:
Memberikan DNS server ke client.

```powershell
Set-DhcpServerV4Scope -ScopeId 192.168.56.0 -LeaseDuration ([TimeSpan]::FromDays(365))
```
Penjelasan:
Durasi peminjaman IP.

---

## 4. DNS
### Fungsi
Menerjemahkan nama domain ke IP.

### Kenapa penting
Tanpa DNS, user harus hafal IP. Itu bukan 1995.

### Command
```powershell
Install-WindowsFeature DNS -IncludeManagementTools
```
Penjelasan:
Mengaktifkan layanan DNS.

```powershell
Add-DnsServerPrimaryZone -NetworkId "192.168.56.0/24"
```
Penjelasan:
Membuat reverse lookup zone (IP → nama).

```powershell
Add-DnsServerResourceRecordPtr -Name "10" -ZoneName "56.168.192.in-addr.arpa" -PtrDomainName "www.a3n4.com"
```
Penjelasan:
Menghubungkan IP ke nama domain.

```powershell
Add-DnsServerResourceRecordA -Name "www" -ZoneName "a3n4.com" -IPv4Address "192.168.56.10"
```
Penjelasan:
Mapping nama ke IP.

---

## 5. JOIN DOMAIN
### Fungsi
Menghubungkan client ke domain.

### Kenapa penting
Tanpa ini, client cuma “tamu”.

### Langkah
```
sysdm.cpl → change → domain → a3n4.com
```
Penjelasan:
Mengubah keanggotaan komputer dari workgroup ke domain.

---

## 6. USER, GROUP, OU
### Fungsi
Manajemen user dan hak akses.

### Command
```powershell
New-ADOrganizationalUnit -Name "a3n4_Staff" -Path "DC=a3n4,DC=com"
```
Penjelasan:
Membuat struktur organisasi di AD.

```powershell
New-ADGroup -Name "HR_Staff" -GroupScope Global -GroupCategory Security
```
Penjelasan:
Membuat group untuk hak akses.

```powershell
New-ADUser -Name "Ammar" -SamAccountName "Ammar" -Enabled $true
```
Penjelasan:
Membuat user baru.

```powershell
Add-ADGroupMember -Identity "HR_Staff" -Members "Ammar"
```
Penjelasan:
Menambahkan user ke group.

---

## 7. IIS
### Fungsi
Menjalankan web server.

### Command
```powershell
Install-WindowsFeature Web-Server -IncludeManagementTools
```
Penjelasan:
Menginstall IIS.

```powershell
New-Website -Name "a3n4web" -Port 80 -HostHeader "a3n4.com"
```
Penjelasan:
Membuat website baru.

---

## 8. DFS
### Fungsi
File sharing terpusat.

### Command
```powershell
Install-WindowsFeature Fs-Dfs-NameSpace
```
Penjelasan:
Mengaktifkan DFS namespace.

```powershell
New-DfsnRoot -Path "\\a3n4.com\data_a3n4"
```
Penjelasan:
Membuat root DFS.

---

## 9. FSRM
### Fungsi
Mengatur limit storage.

### Command
```powershell
New-FsrmQuota -Path "C:\Shared\AllStaff" -Template "250MB"
```
Penjelasan:
Membatasi ukuran folder.

---

## 10. REMOTE DESKTOP
### Fungsi
Akses server jarak jauh.

### Penjelasan
Mengaktifkan koneksi remote berbasis GUI ke server.

---

## 11. ADFS
### Fungsi
Single Sign-On.

### Command
```powershell
Install-WindowsFeature ADFS-Federation
```
Penjelasan:
Menginstall layanan federasi.

---

## 12. FTP
### Fungsi
Transfer file.

### Command
```powershell
New-WebFtpSite -Name "A3N4FTP"
```
Penjelasan:
Membuat FTP server.

---

## 13. MAIL SERVER
### Fungsi
Mengelola email internal.

### Command
```powershell
Add-DnsServerResourceRecordA -Name "mail"
```
Penjelasan:
Mendaftarkan mail server di DNS.

---

## 14. VPN
### Fungsi
Akses jaringan secara aman dari luar.

### Command
```powershell
Install-RemoteAccess -VpnType VPN
```
Penjelasan:
Mengaktifkan VPN server.

---

## 15. NTP
### Fungsi
Sinkronisasi waktu.

### Command
```powershell
w32tm /resync
```
Penjelasan:
Memaksa sinkronisasi waktu dengan server referensi.

---

## Penutup
Sekarang tiap command ada alasan.  
Kalau masih bingung, berarti bukan dokumentasinya yang kurang.

```