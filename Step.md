# Ipset and other early step:
## Function
### SConfig (IP Set) - Menentukan Koordinat Dasar
**Filosofi**: Sebelum membangun gedung (Service), kita harus menentukan koordinat tanah yang tetap.

**Kenapa Server harus Static?** Karena server adalah pusat layanan. Jika alamat IP server berubah-ubah (DHCP), maka klien tidak akan pernah bisa menemukan di mana "kantor" pusatnya.

## Open SConfig
Masuk ke dalam menu setting config:
```command prompt
SConfig
```

jika anda ingin sconfig jalan setiap kali sign in, ketik:
```powershell
set-sconfig -autoLaunch $true
```

## Langkah2 pada menu SConfig
1. pilih 2 untuk mengubah hostname, dan isi sesuai yang diinginkan.

2. pilih 3 untuk setting network, masukkan index ip, kemudian pilih 1 untuk set ip address, selanjutnya ketik S untuk static dan masukkan:
    ```ip
    192.168.56.10
    ```
    kemudian pada subnet tekan enter untuk pilih default 255.255.255.0, dan default gateway kosong, kemudian enter.   
    <!-- then enter, after that chose 2 to set dns preferred and insert 
    dns = 192.168.56.10 -->

3. pilih 13 untuk restart komputer


# Adds:
## Function
### ADDS (Active Directory) - Membentuk Pemerintahan
**Filosofi**: Ini adalah momen kita mendirikan balai kota dan sistem kependudukan.

**Kenapa harus ada ini?** Tanpa AD DS, komputer-komputer hanyalah individu yang asing satu sama lain. AD DS memberikan "identitas" resmi bagi user dan komputer.

## Steps Create
Langkah ini digunakan untuk mengubah Windows Server biasa menjadi Domain Controller (DC) yang mengelola identitas terpusat.
1. Instalasi Role AD DS

    ```PowerShell
    Install-windowsfeature -name AD-Domain-Services -includeManagementTools
    ```
    * Penjelasan: Mengunduh biner dan file sistem yang diperlukan untuk menjalankan layanan Active Directory.

    * Parameter: -includeManagementTools memasang konsol administrasi seperti AD Users and Computers dan modul PowerShell AD agar server bisa dikelola setelah instalasi.

2. Konfigurasi Forest Baru

    ```PowerShell
    Install-ADDSForest -domainName "A3N4.com" -domainNetBiosName "A3N4" -safeModeAdministratorPassword (Convertto-SecureString "Aclsdmin123" -asplaintext -force)
    ```
    * Penjelasan: Perintah ini melakukan "promosi" server menjadi Domain Controller pertama di dalam Forest baru.

    * Parameter Penting:

        * `-domainName`: Menentukan nama DNS domain (FQDN).

        * `-domainNetBiosName`: Nama domain versi lama untuk kompatibilitas.

        * `-safeModeAdministratorPassword`: Mengatur password untuk DSRM (Directory Services Restore Mode), yang digunakan untuk perbaikan jika database AD rusak.`

## Steps Verification
Gunakan perintah ini setelah server melakukan restart otomatis untuk memastikan domain sudah aktif.

1. Verifikasi Instalasi Fitur
    ```PowerShell
    Get-WindowsFeature AD-Domain-Services
    ```
    Fungsi: Memastikan bahwa status fitur sudah tercentang/terpasang ([X]) di sistem operasi.

2. Verifikasi Status Database AD
    ```PowerShell
    Get-Service NTDS
    ```
    Fungsi: Memeriksa layanan NTDS (NT Directory Services). Ini adalah "jantung" dari Active Directory. Jika statusnya Running, berarti database identitas domain sedang aktif.

3. Verifikasi Nama Domain Lingkungan
    ```PowerShell
    $env:USERDOMAIN
    ```
    Fungsi: Menampilkan nama domain tempat user yang sedang login berada. Jika hasilnya adalah A3N4, berarti server sudah sukses mengenali dirinya sebagai bagian dari domain tersebut.

# DHCP 
## Function
### DHCP - Bagian Administrasi Pendaftaran
**Filosofi**: Memberikan nomor rumah secara otomatis kepada pendatang baru (klien).

**Kenapa ini penting?** Agar admin tidak perlu mendatangi setiap meja karyawan untuk mengatur IP manual. DHCP memastikan semua orang dapat "nomor" yang benar dan tidak bentrok.

## Steps Create
1. Instalasi Role DHCP
    ```PowerShell
    Install-WindowsFeature DHCP -includeManagementTools
    ```
    * Penjelasan: Mengunduh dan memasang role DHCP Server di Windows Server.
    * Parameter: `-includeManagementTools` memastikan bahwa GUI (DHCP Console) dan modul PowerShell DHCP ikut terinstal agar server bisa dikelola.

2. Otorisasi DHCP di Active Directory
    ```PowerShell
    Add-DHCPServerInDC -DNSName "a3n4.com" -IpAddress 192.168.56.10
    ```
    * Penjelasan: Mendaftarkan (mengotorisasi) server DHCP ke dalam Active Directory.
    * Tujuan: Mencegah adanya "Rogue DHCP Server" (server DHCP liar). Jika tidak diotorisasi, layanan DHCP tidak akan mau berjalan di lingkungan domain.

3. Membuat Scope Baru
    ```PowerShell
    Add-DhcpServerV4Scope -Name "Scope-a3n4" -StartRange 192.168.56.11 -EndRange 192.168.56.20 -SubnetMask 255.255.255.0 -State Active
    ```
    * Penjelasan: Menentukan rentang alamat IP yang akan dipinjamkan ke komputer klien.
    * Detail: Kamu menyediakan 10 IP (dari .11 sampai .20) dengan status langsung aktif (-State Active).

4. Konfigurasi Gateway (Option 003)
    ```PowerShell
    Set-DHCPServerV4OptionValue -ScopeId 192.168.56.0 -Router 192.168.56.1
    ```
    * Penjelasan: Memberitahu klien alamat IP Default Gateway agar mereka bisa terhubung ke internet atau jaringan lain di luar subnet.

5. Konfigurasi DNS & Nama Domain (Option 006 & 015)
    ```PowerShell
    Set-DHCPServerV4OptionValue -ScopeId 192.168.56.0 -DnsServer 192.168.56.10 -DnsDomain "a3n4.com"
    ```
    * Penjelasan: Memberikan informasi server DNS (192.168.56.10) agar klien bisa menerjemahkan nama domain (seperti https://www.google.com/search?q=google.com) menjadi IP, serta menetapkan sufiks domain utama (a3n4.com).

6. Mengatur Masa Sewa (Lease Duration)
    ```PowerShell
    Set-DHCPServerV4Scope -ScopeId 192.168.56.0 -LeaseDuration ([TimeSpan]::FromDays(365))
    ```
    * Penjelasan: Menentukan berapa lama klien boleh memegang alamat IP tersebut sebelum harus meminta izin lagi.
    * Catatan: Kamu mengaturnya menjadi 365 hari. Ini sangat lama; biasanya untuk jaringan lokal standarnya adalah 8 hari, kecuali untuk perangkat yang sangat jarang berpindah.

## Steps Install & Connect Client
1. Install Windows 10 dan sampai berhasil
2. Set untuk adapter VM client dan juga server:
    * internal network
    * nama: a3n4.com

3. Buka cmd dan ketik:
    ```cmd
    ipconfig
    ```
4. Untuk memastikan ip diset secara otomatis melalui server, ikuti langkah-langkah berikut:
    1. tekan win+r lalu masukkan ncpa.cpl
    2. klik kanan pada ethernet, lalu pilih properties
    4. jika muncul jendela UAC, masukkan username password administrator
    5. klik 2 kali pada baris internet protocol v4
    6. pada jendela ip v4 pilih 
        - obtain an ip address automatically
        - obtain DNS server address automatically
    4. klik ok lalu ok lagi
5. Buka cmd atau powershell dan ketik:
    ```powershell
    ping a3n4.com
    ping 192.168.56.10
    ```

## Steps Verification
1. Verifikasi Status Fitur & Service   
Untuk memastikan role sudah terinstal dan layanannya benar-benar sedang berjalan (Running).

    * Cek Fitur Windows:

        ```PowerShell
        Get-WindowsFeature DHCP
        ```
        Pastikan statusnya sudah [X] Installed.
        
    * Cek Status Service:

        ```PowerShell
        Get-Service DHCPServer
        ```
        Tips: Jika statusnya Stopped, server tidak akan memberikan IP meski konfigurasi sudah benar.

2. Verifikasi Konfigurasi Option (Gateway & DNS)   
Tadi kamu sudah mengatur Router dan DnsServer. Untuk memastikan pengaturan tersebut sudah tersimpan di dalam scope:

    * Cek Option yang Aktif:

        ```PowerShell
        Get-DhcpServerV4OptionValue -ScopeId 192.168.56.0
        ```
        Ini akan menampilkan daftar DNS, Gateway, dan Domain yang sudah kamu input tadi.

3. Verifikasi Otorisasi di Active Directory
Untuk memastikan server kamu sudah diizinkan beroperasi di dalam domain:

    * Cek Daftar Server Terotorisasi:

        ```PowerShell
        Get-DhcpServerInDC
        ```

4. Verifikasi Statistik & Penggunaan IP
Jika nanti sudah ada perangkat yang terhubung, kamu bisa memantau berapa IP yang tersisa dan siapa saja yang sudah meminjam IP.

    * Cek Statistik Scope (Persentase penggunaan):

        ```PowerShell
        Get-DhcpServerV4ScopeStatistics -ScopeId 192.168.56.0
        ```

    * Cek Daftar Klien yang Mendapat IP:

        ```PowerShell
        Get-DhcpServerV4Lease -ScopeId 192.168.56.0
        ```

## Steps Remove
1. Hapus Konfigurasi DHCP (Scope & Otorisasi)
Langkah pertama adalah menghapus scope yang sudah dibuat dan mencabut otorisasi server dari Active Directory.

    * Menghapus Scope:
        Gunakan -ScopeId (alamat network-nya).

        ```powershell
        Remove-DhcpServerV4Scope -ScopeId 192.168.56.0 -Force
        ```

    * Mencabut Otorisasi Server di DC:

        ```powershell
        Uninstall-WindowsFeature DHCP -IncludeManagementTools
        ```
2. Hapus Fitur (Role) DHCP
Setelah konfigurasi bersih, kamu bisa menghapus role DHCP dari Windows Server.   
    
    Uninstall Role dan Tools:

    ```powershell
    Remove-DhcpServerInDC -DNSName "a3n4.com" -IpAddress 192.168.56.10
    ```
3. Restart Server (Penting)
Menghapus role Windows biasanya memerlukan restart agar pembersihan file sistem selesai sepenuhnya. Kamu bisa melakukannya via PowerShell:

    ```powershell
    Restart-Computer
    ```

# DNS 
## Function
### DNS - Buku Telepon Kota
**Filosofi**: Manusia sulit menghafal koordinat (IP), kita lebih mudah hafal nama (www.a3n4.com).

**Kenapa ini penting?** DNS adalah penghubung. Tanpa DNS, komputer klien tidak akan pernah tahu kalau nama domain a3n4.com itu sebenarnya berada di alamat 192.168.56.10.

## Steps Create 
Langkah ini fokus pada instalasi role DNS serta pembuatan Reverse Lookup Zone dan Record (data host).

1. Instalasi Role DNS

    ```PowerShell
    Install-WindowsFeature -Name DNS -IncludeManagementTools
    ```
    Penjelasan: Mengaktifkan layanan DNS Server pada Windows. Sama seperti DHCP, -IncludeManagementTools dipasang agar kamu punya akses ke DNS Manager (GUI) dan perintah PowerShell terkait.

2. Membuat Reverse Lookup Zone

    ```PowerShell
    Add-DnsServerPrimaryZone -NetworkId "192.168.56.0/24" -ZoneFile "56.168.192.in-addr.arpa.dns"
    ```
    Penjelasan: Membuat zona pencarian terbalik. Jika biasanya DNS mencari IP dari nama (Forward), zona ini memungkinkan server mencari nama berdasarkan alamat IP. Sangat berguna untuk keamanan dan diagnosa jaringan.

3. Membuat Record PTR (Pointer)

    ```PowerShell
    Add-DnsServerResourceRecordPtr -Name "10" -ZoneName "56.168.192.in-addr.arpa" -PtrDomainName "www.a3n4.com"
    ```
    Penjelasan: Menambahkan data ke dalam Reverse Lookup Zone tadi. Ini menyatakan bahwa IP akhir .10 (dari network 192.168.56.0) adalah milik www.a3n4.com.

4. Membuat Record A (Address)

    ```PowerShell
    DnsCmd /recordAdd a3n4.com www A 192.168.56.10

    Add-DnsServerResourceRecordA -Name "www" -ZoneName "a3n4.com" -Ipv4Address "192.168.56.10
    ```
    Penjelasan:     
                - `Add-DnsServerResourceRecordA`: Fungsi utama untuk menambah rekaman alamat IPv4 (Record A).   
                - `-Name "www"`: Menentukan nama host atau subdomain yang ingin dibuat.     
                - `-ZoneName "a3n4.com"`: Nama domain utama tempat rekaman ini akan disimpan.     
                - `-IPv4Address "192.168.56.10"`: Alamat IP tujuan yang akan dituju saat seseorang memanggil nama www.a3n4.com.    

## Verification Steps
Setelah dikonfigurasi, kamu harus memastikan server mengenali zona tersebut dan klien bisa menjangkaunya.

1. Verifikasi di Sisi Server
    * Cek Daftar Zone:

        ```PowerShell
        Get-DnsServerZone
        ```
        Memastikan zona a3n4.com dan zona in-addr.arpa (Reverse) berstatus "Primary" dan aktif.

    * Cek Isi Record:

        ```PowerShell
        Get-DnsServerResourceRecord -ZoneName "a3n4.com"
        ```
        Melihat apakah nama www sudah muncul di daftar beserta IP-nya.

2. Verifikasi di Sisi Klien
    * Ping Test:

        ```PowerShell
        ping www.a3n4.com
        ```
        Jika berhasil, berarti sistem operasi sudah bisa mengenali nama tersebut.

    * Lookup Test:

        ```PowerShell
        nslookup www.a3n4.com
        ```
        Ini adalah tes murni DNS. Ia akan menampilkan server mana yang menjawab dan IP berapa yang diberikan. Ini cara paling akurat untuk memastikan DNS bekerja.

# Join Domain
## Function
### Join Domain - Menjadi Warga Negara Resmi
**Filosofi**: Setelah jalurnya ada (IP), namanya terdaftar (DNS), dan dapet nomor urut (DHCP), barulah klien mendaftarkan diri menjadi "Warga Negara" resmi di bawah naungan Server.

**Kenapa ini langkah terakhir?** Karena untuk Join Domain, klien butuh jalur komunikasi yang matang. Jika DNS atau IP salah, klien tidak akan pernah bisa "mengetuk pintu" Server.

### Why
Join Domain adalah proses memasukkan komputer ke dalam sebuah "keluarga" atau sistem manajemen terpusat. Berikut gunanya:

1. Satu Akun untuk Semua Komputer (Single Sign-On)   
Tanpa Domain, kamu harus buat akun di tiap PC. Dengan Domain, kamu cukup buat satu user di Server (Active Directory), dan user itu bisa login di PC mana pun yang sudah join domain.

2. Pengaturan Terpusat (Group Policy / GPO)   
Ini yang paling sakti. Kamu bisa mengatur semua komputer dari satu layar Server. Contohnya:
Melarang semua karyawan ganti wallpaper.
Mematikan fungsi USB Flashdisk di semua PC agar aman dari virus.
Otomatis install aplikasi ke 100 komputer sekaligus tanpa mendatangi PC-nya satu per satu.

3. Keamanan & Hak Akses   
Jika kamu punya File Server (folder sharing), kamu bisa mengatur dengan sangat detail: "Hanya divisi Keuangan yang boleh buka folder Gaji, divisi lain tidak bisa klik sama sekali." Ini jauh lebih aman daripada sharing folder biasa (Workgroup).

4. Inventori & Monitoring   
Server akan mencatat semua komputer yang tergabung. Kamu jadi tahu komputer mana yang sedang aktif, siapa yang sedang login, dan kapan terakhir mereka ganti password.  

Gampangnya begini:
DNS & DHCP = Jalur kabel dan pemberian nomor rumah agar bisa saling panggil.
Join Domain = Aturan hukum dan sistem kependudukan agar semuanya patuh pada satu pimpinan (Server).

## Steps Create
1. Login Windows Client (Pastikan Windows Pro)
2. Klik win+r lalu masukkan `sysdm.cpl` untuk masuk ke system properties
3. pilih change dibagian bawah kanan
4. Pilih Domain lalu masukkan nama domainnya
5. Klik ok maka muncul window untuk meminta masukkan credential
6. Masukkan username dan password domain untuk memberi izin, lalu klik ok
7. Restart Client

## Steps Verification
1. Login Windows Client
2. Klik win+r lalu masukkan `ncpa.cpl` 
3. klik dua kali pada ethernet
4. Pada status ethernet window klik details
5. Pastikan client sudah menunjuk ip address ke dhcp dan dns server ke domain

# User & Group
## Function
Jika Domain adalah sebuah kantor besar, maka:

1. Organizational Unit (OU): Ibarat sebuah Gedung atau Ruangan. Gunanya untuk merapikan database AD agar tidak berantakan. Tanpa OU, semua user akan tercampur di satu folder besar.

2. User: Ibarat Kunci Masuk. Setiap karyawan punya kunci (username/password) unik. Ini memastikan kita tahu "siapa melakukan apa" di dalam jaringan.

3. Group: Ibarat Akses Khusus. Daripada memberikan izin satu per satu ke 100 orang, kita cukup masukkan 100 orang itu ke kelompok "Finance", lalu kelompok "Finance" kita beri akses ke folder gaji. Jauh lebih cepat dan efisien.

## Steps Create
1. Membuat Organizational Unit (OU)

    ```PowerShell
    New-AdOrganizationalUnit -Name "a3n4_Staff" -Path "DC=a3n4, DC=com"
    ```
    * Penjelasan: Membuat wadah bernama a3n4_Staff.

    * Parameter: -Path menjelaskan di mana wadah ini berada (di dalam root domain a3n4.com).

2. Membuat Security Group

    ```PowerShell
    # Contoh
    New-AdGroup -Name "HR_Staff" -GroupScope Global -Path "OU=a3n4_Staff, DC=a3n4, DC=com" -GroupCategory Security

    # Isi
    New-AdGroup -Name "All_Staff" -GroupScope Global -Path "OU=a3n4_Staff, DC=a3n4, DC=com" -GroupCategory Security
    New-AdGroup -Name "Finance_Staff" -GroupScope Global -Path "OU=a3n4_Staff, DC=a3n4, DC=com" -GroupCategory Security
    New-AdGroup -Name "HR_Staff" -GroupScope Global -Path "OU=a3n4_Staff, DC=a3n4, DC=com" -GroupCategory Security
    New-AdGroup -Name "Marketing_Staff" -GroupScope Global -Path "OU=a3n4_Staff, DC=a3n4, DC=com" -GroupCategory Security
    ```
    * Penjelasan: Membuat kelompok departemen.

    * Parameter: -GroupScope Global berarti grup ini bisa digunakan di seluruh domain, dan -GroupCategory Security berarti grup ini bisa digunakan untuk mengatur hak akses folder/file.

3. Membuat Akun User

    ```PowerShell
    # Contoh 
    New-ADUser -Name "Ammar" -SamAccountName "Ammar" -AccountPassword (...) -Enabled $true

    # Isi
    # Ammar
    New-ADUser -Name "Ammar" -GivenName "Ammar" -Surname "" -SamAccountName "Ammar" -UserPrincipalName "Ammar@a3n4.com" -Path "OU=a3n4_Staff,DC=a3n4,DC=com" -AccountPassword (ConvertTo-SecureString "##ammarTi2b" -AsPlainText -Force) -Enabled $true

    #Nailis Saputri
    New-AdUser -Name "Nailis Saputri" -GivenName "Nailis" -Surname "Saputri" -SamAccountName "NailisSaputri" -UserPrincipalName "NailisSaputri@a3n4.com" -Path "OU=a3n4_Staff, DC=a3n4, DC=com" -AccountPassword (ConvertTo-SecureString "##nailisTi2b" -AsPlainText -Force) -Enabled $true

    # Dhiyaul Atha
    New-ADUser -Name "Dhiyaul Atha" -GivenName "Dhiyaul" -Surname "Atha" -SamAccountName "DhiyaulAtha" -UserPrincipalName "DhiyaulAtha@a3n4.com" -Path "OU=a3n4_Staff,DC=a3n4,DC=com" -AccountPassword (ConvertTo-SecureString "##athaTi2b" -AsPlainText -Force) -Enabled $true

    # Naiza Fitri
    New-ADUser -Name "Naiza Fitri" -GivenName "Naiza" -Surname "Fitri" -SamAccountName "NaizaFitri" -UserPrincipalName "NaizaFitri@a3n4.com" -Path "OU=a3n4_Staff,DC=a3n4,DC=com" -AccountPassword (ConvertTo-SecureString "##naizaTi2b" -AsPlainText -Force) -Enabled $true

    # Nayla Mutia
    New-ADUser -Name "Nayla Mutia" -GivenName "Nayla" -Surname "Mutia" -SamAccountName "NaylaMutia" -UserPrincipalName "NaylaMutia@a3n4.com" -Path "OU=a3n4_Staff,DC=a3n4,DC=com" -AccountPassword (ConvertTo-SecureString "##naylaTi2b" -AsPlainText -Force) -Enabled $true

    # Nikmal Wakil
    New-ADUser -Name "Nikmal Wakil" -GivenName "Nikmal" -Surname "Wakil" -SamAccountName "NikmalWakil" -UserPrincipalName "NikmalWakil@a3n4.com" -Path "OU=a3n4_Staff,DC=a3n4,DC=com" -AccountPassword (ConvertTo-SecureString "##nikmalTi2b" -AsPlainText -Force) -Enabled $true

    # Arini Safitri
    New-ADUser -Name "Arini Safitri" -GivenName "Arini" -Surname "Safitri" -SamAccountName "AriniSafitri" -UserPrincipalName "AriniSafitri@a3n4.com" -Path "OU=a3n4_Staff,DC=a3n4,DC=com" -AccountPassword (ConvertTo-SecureString "##ariniTi2b" -AsPlainText -Force) -Enabled $true
    ```
    * Penjelasan: Mendaftarkan individu ke dalam sistem.

    * Parameter Penting:

        * -SamAccountName: Nama yang digunakan untuk login (misal: Ammar).

        * -UserPrincipalName: Format login email (Ammar@a3n4.com).

        * -AccountPassword: Memberikan password awal yang sudah di-enkripsi.

        * -Enabled $true: Memastikan akun langsung aktif dan bisa digunakan untuk login.

4. Aktivasi User (Opsional)

    ```PowerShell
    # Jika tidak memakai -Enabled $true maka harus mengaktifkan user secara manual dengan perintah berikut:
    Enable-ADAccount -Identity "Ammar" 
    # Atau untuk seluruh user dengan satu command:
    Get-ADUser -Filter * -SearchBase "OU=a3n4_Staff,DC=a3n4,DC=com" | Enable-ADAccount
    ```
    Penjelasan: Perintah borongan untuk mengaktifkan semua user yang ada di dalam OU tersebut jika saat pembuatan lupa menambahkan -Enabled $true.

5. Memasukkan User ke Group (Membership)

    ```PowerShell
    Add-AdGroupMember -Identity "Nama_Group" -Members "Nama_User"
    ```
    Penjelasan: Menghubungkan user dengan departemennya. Setelah ini dijalankan, user tersebut otomatis mendapatkan semua hak akses yang dimiliki oleh grup tersebut.

## Verification Step
Verifikasi Anggota Group
```PowerShell
Get-ADGroupMember -Identity "All_Staff"
```
Fungsi: Menampilkan daftar siapa saja yang sudah berhasil masuk ke dalam grup All_Staff. Jika nama-nama user yang kamu buat tadi muncul di sini, berarti konfigurasi berhasil.

Tips Wit: Ingat, memberikan password yang sama ke semua user (##ammarTi2b, dll) itu memudahkan saat praktik, tapi di dunia nyata, biasanya kita pakai opsi ChangePasswordAtLogon supaya mereka ganti password sendiri saat pertama kali masuk!
Contoh:
```powershell
New-ADUser -Name "Arini Safitri" -GivenName "Arini" -Surname "Safitri" -SamAccountName "AriniSafitri" -UserPrincipalName "AriniSafitri@a3n4.com" -Path "OU=a3n4_Staff,DC=a3n4,DC=com" -AccountPassword (ConvertTo-SecureString "##ariniTi2b" -AsPlainText -Force) -Enabled $true -ChangePasswordAtLogon $true
```
# IIS
## Function
### IIS (Internet Information Services) - Membangun Toko Online
**Filosofi**: Setelah semua infrastruktur siap, sekarang saatnya kita membuka "toko online" agar dunia bisa mengakses layanan yang kita tawarkan.
**Kenapa ini penting?** Tanpa IIS, server kamu hanyalah komputer biasa. Dengan IIS, kamu bisa membuat website, aplikasi web, atau layanan online yang bisa diakses dari mana saja.

## Steps Create
1. Instalasi Role IIS

    ```PowerShell
    Install-WindowsFeature -Name Web-Server -IncludeManagementTools
    ```
    Penjelasan: Mengaktifkan layanan IIS pada Windows Server. Parameter -IncludeManagementTools memastikan kamu mendapatkan akses ke IIS Manager (GUI) dan modul PowerShell terkait.

2. Membuat Folder untuk Website
    ```Powershell
    New-Item -Path "C:\inetpub\a3n4web" -ItemType directory
    ```
    Penjelasan: 

3. Membuat Website Baru
    ```Powershell
    New-Website -Name "a3n4web" -Port 80 -IpAddress "*" -HostHeader "a3n4.com" -PhysicalPath "C:\inetpub\a3n4web"
    ```
    Penjelasan:


```Powershell
Copy-Item -Path "" -Destination "C:\inetpub\a3n4web" -Recurse

Install-WindowsFeature Web-Static-Content

New-SelfSignedCertificate -DnsName "a3n4.com" -CertStoreLocation "cert:\LocalMachine\My"

Import-Module WebAdministration

New-WebBinding -Name "a3n4web" -Protocol https -port 443 -IpAddress "*" -HostHeader "a3n4.com"

Get-Item "cert:\LocalMachine\My\<ThumbPrint>" | New-Item "IIS:SslBindings\0.0.0.0!443!a3n4.com"

```
## Steps Verification
1. Login Client
2. Ketik http://a3n4.com untuk masuk ke IIS web
3. Maka akan berhasil ke web
4. Ketik https://a3n4.com untuk masuk ke IIS Certificate
5. Maka akan tampil "Your Connection Isn't Private"
6. Klik Advanced lalu continue to a3n4.com (unsafe)
7. Maka berhasil masuk ke IISC


# Remote Desktop
## Function


## Steps Create
### Server
1. Masuk ke SConfig
2. Pilih 7
3. Pilih E untuk Enabled Remote Dekstop
4. Pilih 1 untuk more secure
5. Enter dan pilih 13

### Client
1. Login Client
2. Buka Search bar atau win+r
3. Cari Remote Dekstop Connection atau mstsc
4. Pada Window Remote Dekstop
5. Isi Nama Computer a3n4.com
6. Pada jendela credential masukkan password user 
7. Muncul Warning remote computer cannot be verified klik ok (Karna ini server buatan sendiri jadi aman)

## Steps Verification


# ADFS
## Steps Create
```Powershell
Install-WindowsFeature ADFS-Federation -IncludeManagementTools

Add-DnsServerResourceRecordA -Name "adfs" -ZoneName "a3n4.com" -Ipv4Address 192.168.56.10

New-SelfSignedCertificate -DnsName "adfs.a3n4.com" -CertStoreLocation "cert:\LocalMachine\My"

Add-KdsRootKey -EffectiveTime ((Get-Date).AddDays(1))

$pass = "2025" | ConvertTo-SecureString -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential ("A3N4\administrator", $pass)

Install-ADFSFarm -CertificateThumbprint "8A5421340313C07AA2017F76F55984BAC754C422" -FederationServiceName "adfs.a3n4.com" -FederationServiceDisplayName "A3N4 Federation Service" -ServiceAccountCredential $cred

# Masukkan username akun dan password yang akan digunakan lalu klik ok

Set-ADFSProperties -EnableIdpInitiatedSignOnPage $true

Get-Service ADFSSRV

### PLUS Identity
New-AdfsWebTheme -Name "TemaBaru" -SourceName "Default"

Set-AdfsWebTheme -TargetName "TemaBaru" -Illustration @{path="Z:\wall_a3n4.jpg"}

Set-AdfsWebTheme -TargetName "TemaBaru" -Logo @{path="Z:\logo_a3n4.png"}

Set-AdfsWebConfig -ActiveThemeName "TemaBaru"
```

### Client
1. Login Client
2. Pada Browser Masukkan https://adfs.a3n4.com/adfs/ls/IdpInitiatedSignOn.aspx
3. Tampil "Your connection is not private"
4. Pilih advance dan continue to the web
5. Maka akan tampil halaman dari web adfs
6. Klik Sign in untuk login
7. Masukkan Username dan password lalu sign in



# DFS
## Steps Install
```Powershell
Install-WindowsFeature Fs-Dfs-NameSpace -IncludeManagementTools

New-Item -Path "C:\DFSRoots\a3n4_shared" -ItemType Directory

New-SmbShare -Name "DFSa3n4" -Path "C:\DFSRoots\a3n4_shared" -FullAccess "a3n4\Administrator"

New-Dfsnroot -Type domainv2 -Path "\\a3n4.com\data_a3n4" -TargetPath "\\a3n4_server\DFSa3n4"

New-Item -ItemType Directory -Path "C:\Shared"

New-SmbShare -Name "data_share" -Path "C:\Shared" -FullAccess "a3n4\Administrator"

New-Item -ItemType Directory -Path "C:\Shared\AllStaff"
New-Item -ItemType Directory -Path "C:\Shared\HRStaff"
New-Item -ItemType Directory -Path "C:\Shared\MarketingStaff"
New-Item -ItemType Directory -Path "C:\Shared\FinanceStaff"

Icacls "C:\Shared\AllStaff"
Icacls "C:\Shared\AllStaff" /inheritance:r
Icacls "C:\Shared\AllStaff" /grant "a3n4\AllStaff:(OI)(CI)M" "a3n4\Administrator:(OI)(CI)F"
Icacls "C:\Shared\AllStaff"

Icacls "C:\Shared\HRStaff" 
Icacls "C:\Shared\HRStaff" /inheritance:r
Icacls "C:\Shared\HRStaff" /grant "a3n4\HRStaff:(OI)(CI)M" "a3n4\Administrator:(OI)(CI)F"
Icacls "C:\Shared\HRStaff"

Icacls "C:\Shared\FinanceStaff"
Icacls "C:\Shared\FinanceStaff" /inheritance:r
Icacls "C:\Shared\FinanceStaff" /grant "a3n4\FinanceStaff:(OI)(CI)M" "a3n4\Administrator:(OI)(CI)F"
Icacls "C:\Shared\FinanceStaff"

Icacls "C:\Shared\MarketingStaff" /inheritance:r
Icacls "C:\Shared\MarketingStaff" /grant "a3n4\MarketingStaff:(OI)(CI)M" "a3n4\Administrator:(OI)(CI)F"
Icacls "C:\Shared\MarketingStaff"

New-DfsnFolder -Path "\\a3n4.com\data_a3n4\AllStaff" -TargetPath "\\a3n4_server\data_share\AllStaff"
New-DfsnFolder -Path "\\a3n4.com\data_a3n4\HRStaff" -TargetPath "\\a3n4_server\data_share\HRStaff"
New-DfsnFolder -Path "\\a3n4.com\data_a3n4\MarketingStaff" -TargetPath "\\a3n4_share\data_a3n4\MarketingStaff"
New-DfsnFolder -Path "\\a3n4.com\data_a3n4\FinanceStaff" -TargetPath "\\a3n4_server\data_share\FinanceStaff"

Grant-SmbShareAccess -Name DFSa3n4 -Account "a3n4\All_Staff" -AccessRight Read -Force
Grant-SmbShareAccess -Name DFSa3n4 -Account "a3n4\HR_Staff" -AccessRight Read -Force
Grant-SmbShareAccess -Name DFSa3n4 -Account "a3n4\Finance_Staff" -AccessRight Read -Force
Grant-SmbShareAccess -Name DFSa3n4 -Account "a3n4\Marketing_Staff" -AccessRight Read -Force

Grant-SmbShareAccess -Name data_share -Account "a3n4\All_Staff" -AccessRight Change -Force
Grant-SmbShareAccess -Name data_share -Account "a3n4\HR_Staff" -AccessRight Change -Force
Grant-SmbShareAccess -Name data_share -Account "a3n4\Finance_Staff" -AccessRight Change -Force
Grant-SmbShareAccess -Name data_share -Account "a3n4\Marketing_Staff" -AccessRight Change -Force

Set-DfsnRoot -Path "\\a3n4\data_a3n4" -EnableAccessBasedEnumeration $true 

takeown /f "C:\DFSRoots"

Icacls "C:\DFSRoots" /setowner "NT AUTHORITY\SYSTEM" /T /C

Icacls "C:\DFSRoots"

takeown /f "C:\DFSRoots\a3n4_shared\AllStaff" /r /d y
Icacls "C:\DFSRoots\a3n4_shared\AllStaff" /inheritance:r
Icacls "C:\DFSRoots\a3n4_shared\AllStaff" /grant "a3n4\All_Staff:(OI)(CI)M"
Icacls "C:\DFSRoots\a3n4_shared\AllStaff"

takeown /f "C:\DFSRoots\a3n4_shared\HRStaff" /r /d y
Icacls "C:\DFSRoots\a3n4_shared\HRStaff" /inheritance:r
Icacls "C:\DFSRoots\a3n4_shared\HRStaff" /grant "a3n4\All_Staff:(OI)(CI)M"
Icacls "C:\DFSRoots\a3n4_shared\HRStaff"

takeown /f "C:\DFSRoots\a3n4_shared\MarketingStaff" /r /d y
Icacls "C:\DFSRoots\a3n4_shared\MarketingStaff" /grant "a3n4\Marketing_Staff:(OI)(CI)M"
Icacls "C:\DFSRoots\a3n4_shared\MarketingStaff"

takeown /f "C:\DFSRoots\a3n4_shared\FinanceStaff" /r /d y
Icacls "C:\DFSRoots\a3n4_shared\FinanceStaff" /grant "a3n4\Finance_Staff:(OI)(CI)M"
Icacls "C:\DFSRoots\a3n4_shared\FinanceStaff"


```
## Verification Step
1. Login Client
2. Klik kanan pada logo windows pilih run
3. Ketik \\a3n4.com\data_a3n4
4. Maka akan direct ke folder data_a3n4, yang menampilkan folder group yang dimasukin oleh user tsb.
5. Pada Folder Group User tsb buatlah sebuah file
6. Di Server masuk ke directory C:\Shared\<Folder_Group_user_sebelumnya>
7. Maka akan tampil file yang telah dibuat oleh user tsb. 


# FSRM
## Steps Install
```Powershell
Install-WindowsFeature
```

# NTP

# FTP

# VPN


# Mail Server
