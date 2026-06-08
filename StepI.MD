# First
```note
siapkan file berikut
Composer-Setup.exe -> agar laravel dan serve projek
group_manage.sql -> agar db projek
group-manage.zip -> file projek
mysql-8.0.46-winx64.zip -> agar db
node-v22.17.1-x64.msi -> agar npm
php-8.4.12-Win32-vs17-x64.zip -> agar php
rewrite_amd64_en-US.msi -> agar laravel/db
VC_redist.x64.exe -> agar php
web.config -> agar file
```

# Second-php
php extract .zip lalu tambahkan `c:/php` ke path:
```powershell
mkdir c:/php
Expand-Archive -Path "php.zip" -DestinationPath "c:/php"
$path = [System.Environment]::GetEnvironmentVariable("Path", "User")
[System.Environment]::SetEnvironmentVariable("Path", "$path;C:\php" "User")
php -v
```

pastikan ini uncomment:
```ini
extension=fileinfo
extension=mbstring
extension=openssl
extension=pdo_mysql
extension=mysqli
extension=curl
extension=zip
```

# third
mysql extract .zip lalu tambahkan `c:/mysql/bin` ke path:
```powershell
mkdir c:/mysql
Expand-Archive -Path "mysql.zip" -DestinationPath "c:/mysql"
$path = [System.Environment]::GetEnvironmentVariable("Path", "User")
[System.Environment]::SetEnvironmentVariable("Path", "$path;C:\mysql\bin" "User")
mysql --version
```
lalu buat folder `mysql/data` lalu jalankan:
```powershell
mysqld --initialize-insecure --basedir=C:\mysql --datadir=C:\mysql\data
```
lalu buat file `mysql/my.ini` dan isi
```ini
[mysqld]
basedir=C:/MYSQL
datadir=C:/MYSQL/data
port=3306

[client]
port=3306
```

lalu install service mysql:
```powershell
mysqld --install MySQL --defaults-file=C:\mysql\my.ini
Start-Service MySQL
```

setelah itu lanjut import db projek:
```cmd
mysql -u root -e "create database group_manage"
mysql -u root group_manage < z:/iis-laravel/group_manage.sql
```

# forth
Install composer, node, rewrite, vc_redistx64
lalu extract file .zip yang berisi projek ke dalam c:/inetpub
```PowerShell
# untuk run .exe/.msi sentuh saja file executeable nya
# pastikan internet nyala/NAT ketika penginstalan composer
"C:/Composer-Setup.exe"

Expand-Archive -Path "group-manage.zip" -DestinationPath "c:/inetpub/group-manage"
```

lalu install feature windows untuk iis:
```powershell
Install-WindowsFeature -Name Web-Server -IncludeManagementTools
Install-WindowsFeature -Name Web-CGI
Install-WindowsFeature -Name Web-Static-Content
```

lanjut konfigurasi fastcgi:
```powershell
Import-Module WebAdministration
Add-WebConfiguration -Filter /system.webServer/fastCgi -value @{fullpath='C:\php\php-cgi.exe'}
New-WebHandler -Name PHP-FastCGI -Path "*.php" -Verb "*" -Modules FastCgiModule -ScriptProcessor "C:\php\php-cgi.exe"
```

# fifth
setelah itu buat dns record A untuk web:
```powershell
Add-DnsServerResourceRecordA -Name "group-manage" -ZoneName "ammar.com" -IPv4Address "172.18.18.180"
```

lalu buat website:
```powershell
Import-Module WebAdministration
New-WebSite -Name "group-manage" -Port 80 -HostHeader "group-manage.ammar.com" -PhysicalPath "C:\inetpub\group-manage\public"
```

Setelah itu buat webpool untuk pemberian permission kepada web:
```powershell
New-WebAppPool -Name "group-manage"
Set-ItemProperty -Path "IIS:\Sites\group-manage" -Name applicationPool -Value group-manage

#Lanjut pemberian permission
icacls "C:\inetpub\group-manage\storage" /grant "IIS AppPool\group-manage:(OI)(CI)F" /T
```


# sixth
masuk ke dalam folder web:
```powershell
cd C:\inetpub\group-manage
```

lalu buat file `web.config` di dalam folder `public`
```powershell
New-Item -Path "public\web.config" -ItemType "file"
notepad "public\web.config"
```
dan isi:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <defaultDocument>
      <files>
        <add value="index.php" />
      </files>
    </defaultDocument>
    <rewrite>
      <rules>
        <rule name="Laravel" stopProcessing="true">
          <match url=".*" ignoreCase="false" />
          <conditions>
            <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
            <add input="{REQUEST_FILENAME}" matchType="IsDirectory" negate="true" />
          </conditions>
          <action type="Rewrite" url="index.php" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
</configuration>
```

setelah itu sesuaikan .env untuk password dengan mysql di windows server
lanjut jalankan command:
```powershell
cd C:\inetpub\group-manage

composer install
php artisan key:generate
npm install
npm run build

#setelah itu hapus public\hot
Remove-Item public\hot

php artisan config:clear
php artisan cache:clear
php artisan route:clear
php artisan view:clear
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

# seventh
Membuat Binding Https menggunakan self signed certificate
```powershell
# Membuat self signed certificate
$cert = New-SelfSignedCertificate -DnsName "group-manage.ammar.com" -CertStoreLocation "cert:\LocalMachine\My"

# Menambahkan binding HTTPS ke situs IIS
Import-Module WebAdministration
New-WebBinding -Name "group-manage" -Protocol https -Port 443 -HostHeader "group-manage.ammar.com"

# Mengaitkan sertifikat dengan binding HTTPS
New-item -Path "IIS:\SslBindings\0.0.0.0!443!group-manage.ammar.com" -Thumbprint $cert.Thumbprint -SslFlags 1
```


# Verifikasi
jalankan command untuk melihat service web dan database berjalan
```powershell
Get-Service MySQL
Get-Service W3SVC
```

Masuk ke dalam http://group-manage.ammar.com lalu register akun , dan buat group projek
Masuk ke dalam https://group-manage.ammar.com lalu advance setelah warning