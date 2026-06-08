# Windows Server Configuration Guide - A3N4

## 1. IP Setup & Initial Configuration (SConfig)

### Purpose
Set the basic server configuration before building other services.

### Philosophy
Before building any system (Active Directory, DNS, File Server, etc.), the foundation must be stable. A server is like a headquarters — if the address keeps changing, no one will know where to go.

### Steps - Configure

1. Open SConfig:
   ```
   SConfig
   ```

   To auto-launch SConfig at every sign-in:
   ```powershell
   Set-SConfig -AutoLaunch $true
   ```

2. Select **2** to change the hostname, then enter the desired name.

3. Select **3** to configure network settings.
   - Enter the interface index (displayed when selecting this option)
   - Select **1** to set the IP address
   - Type **S** for static, then enter:
     ```
     172.18.18.180
     ```
   - Press Enter for subnet (default: 255.255.255.0)
   - Leave the default gateway empty, then press Enter
   - Select **2** to set the preferred DNS server, then enter:
     ```
     172.18.18.180
     ```

4. Select **13** to restart the computer.

# ADDS

## Function
### ADDS (Active Directory Domain Services) - Establishing the Government

### Philosophy
Without AD DS, computers are isolated strangers to one another. AD DS provides official identities for users and computers, enabling centralized management, authentication, and authorization — just like a government issues IDs to its citizens.

### Steps - Create
```PowerShell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "A3N4.com" -DomainNetBiosName "A3N4" -SafeModeAdministratorPassword (ConvertTo-SecureString "Aclsdmin123" -AsPlainText -Force)
```

### Steps - Remove
```PowerShell
Uninstall-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

### Steps - Verification
```PowerShell
Get-WindowsFeature AD-Domain-Services
Get-Service NTDS
$env:USERDOMAIN
```

# DHCP

## Function
### DHCP (Dynamic Host Configuration Protocol) - Automatic Address Assignment

### Philosophy
DHCP acts as an automatic registration desk. When a new client arrives, DHCP assigns them an IP address so they do not have to wait for manual configuration. This prevents IP conflicts and saves administration time.

### Steps - Create

```PowerShell
Install-WindowsFeature DHCP -IncludeManagementTools
Add-DHCPServerInDC -DNSName "a3n4.com" -IpAddress 172.18.18.180
Add-DhcpServerV4Scope -Name "Scope-a3n4" -StartRange 172.18.18.11 -EndRange 172.18.18.20 -SubnetMask 255.255.255.0 -State Active
Set-DhcpServerV4OptionValue -ScopeId 172.18.18.0 -Router 172.18.18.1
Set-DhcpServerV4OptionValue -ScopeId 172.18.18.0 -DnsServer 172.18.18.180 -DnsDomain "a3n4.com"
Set-DhcpServerV4Scope -ScopeId 172.18.18.0 -LeaseDuration ([TimeSpan]::FromDays(365))
```

### Steps - Client Setup

1. Install Windows 10 on the client VM.
2. Configure both server and client VM adapters:
   - **Internal Network**
   - **Name**: `a3n4.com`
3. Open **cmd** and run:
   ```cmd
   ipconfig
   ```
4. Ensure the client obtains its IP automatically:
   - Press `Win + R`, type `ncpa.cpl`
   - Right-click **Ethernet** → **Properties**
   - If prompted by UAC, enter admin credentials
   - Double-click **Internet Protocol Version 4 (TCP/IPv4)**
   - Select:
     - **Obtain an IP address automatically**
     - **Obtain DNS server address automatically**
   - Click **OK** → **OK**
5. Verify connectivity:
   ```powershell
   ping a3n4.com
   ping 172.18.18.180
   ```

### Steps - Verification

```PowerShell
Get-WindowsFeature DHCP
Get-Service DHCPServer
Get-DhcpServerV4OptionValue -ScopeId 172.18.18.0
Get-DhcpServerInDC
Get-DhcpServerV4ScopeStatistics -ScopeId 172.18.18.0
Get-DhcpServerV4Lease -ScopeId 172.18.18.0
```

### Steps - Remove

```powershell
Remove-DhcpServerV4Scope -ScopeId 172.18.18.0 -Force
Uninstall-WindowsFeature DHCP -IncludeManagementTools
Remove-DhcpServerInDC -DNSName "a3n4.com" -IpAddress 172.18.18.180
Restart-Computer
```

# DNS

## Function
### DNS (Domain Name System) - The Network Phonebook

### Philosophy
Humans find it difficult to remember numbers (IP addresses); we are far better at remembering names. DNS translates human-friendly domain names (like `www.a3n4.com`) into machine-readable IP addresses. Without DNS, clients would never know that `a3n4.com` lives at `172.18.18.180`.

### Steps - Create

```PowerShell
Install-WindowsFeature -Name DNS -IncludeManagementTools

Add-DnsServerPrimaryZone -NetworkId "172.18.18.0/24" -ZoneFile "18.18.172.in-addr.arpa.dns"

Add-DnsServerResourceRecordPtr -Name "180" -ZoneName "18.18.172.in-addr.arpa" -PtrDomainName "www.a3n4.com"

Add-DnsServerResourceRecordA -Name "www" -ZoneName "a3n4.com" -IPv4Address "172.18.18.180"
```

### Steps - Remove

```PowerShell
Remove-DnsServerResourceRecord -Name "www" -ZoneName "a3n4.com" -RRType A -Force
Remove-DnsServerResourceRecord -Name "180" -ZoneName "18.18.172.in-addr.arpa" -RRType PTR -Force
Remove-DnsServerZone -Name "18.18.172.in-addr.arpa" -Force
Remove-WindowsFeature -Name DNS -IncludeManagementTools
```

### Steps - Verification

```PowerShell
Get-DnsServerZone
Get-DnsServerResourceRecord -ZoneName "a3n4.com"
ping www.a3n4.com
nslookup www.a3n4.com
```

# Join Domain

## Function
### Join Domain - Becoming an Official Domain Citizen

### Philosophy
After the network path is ready (IP), the name is registered (DNS), and automatic addressing is in place (DHCP), the client can finally register itself as an official member of the domain. This is the last step because the client needs a fully functional communication channel — if DNS or IP is misconfigured, the client can never "knock on the server's door."

### Steps - Create

1. Log in to the Windows Client (must be Windows Pro/Enterprise).
2. Press `Win + R`, type `sysdm.cpl` to open **System Properties**.
3. Click **Change** (bottom-right section).
4. Select **Domain** and enter the domain name:
   ```
   A3N4.com
   ```
5. Click **OK** — a credential window will appear.
6. Enter domain administrator credentials to authorize the join, then click **OK**.
7. Restart the client when prompted.

### Steps - Remove

1. Log in to the Windows Client.
2. Press `Win + R`, type `sysdm.cpl` to open **System Properties**.
3. Click **Change** (bottom-right section).
4. Select **Workgroup** and enter a workgroup name (e.g., `WORKGROUP`).
5. Click **OK** — a credential window will appear.
6. Enter domain administrator credentials to authorize the removal, then click **OK**.
7. Restart the client when prompted.

### Steps - Verification

1. Log in to the Windows Client.
2. Press `Win + R`, type `ncpa.cpl`.
3. Double-click **Ethernet**.
4. In the **Ethernet Status** window, click **Details**.
5. Verify that:
   - **IPv4 Address** is assigned by DHCP (from the server scope)
   - **DNS Server** points to the domain server (`172.18.18.180`)

# User & Group

## Function
If a domain is a large office, then:

1. **Organizational Unit (OU)** — Like a building or room. It organizes the AD database so everything is not mixed together in one big folder.

2. **User** — Like a key. Each employee has a unique key (username/password). This ensures we know "who did what" on the network.

3. **Group** — Like access badges. Instead of granting permissions one by one to 100 people, we put them into groups (e.g., "Finance") and grant permissions to the group. Much faster and more efficient.

### Steps - Create

1. **Create an Organizational Unit (OU)**

   ```PowerShell
   New-AdOrganizationalUnit -Name "a3n4_Staff" -Path "DC=a3n4, DC=com"
   ```

2. **Create Security Groups**

   ```PowerShell
   New-AdGroup -Name "All_Staff" -GroupScope Global -Path "OU=a3n4_Staff, DC=a3n4, DC=com" -GroupCategory Security
   New-AdGroup -Name "Finance_Staff" -GroupScope Global -Path "OU=a3n4_Staff, DC=a3n4, DC=com" -GroupCategory Security
   New-AdGroup -Name "HR_Staff" -GroupScope Global -Path "OU=a3n4_Staff, DC=a3n4, DC=com" -GroupCategory Security
   New-AdGroup -Name "Marketing_Staff" -GroupScope Global -Path "OU=a3n4_Staff, DC=a3n4, DC=com" -GroupCategory Security
   ```

3. **Create User Accounts**

   ```PowerShell
   New-ADUser -Name "Ammar" -GivenName "Ammar" -Surname "" -SamAccountName "Ammar" -UserPrincipalName "Ammar@a3n4.com" -Path "OU=a3n4_Staff,DC=a3n4,DC=com" -AccountPassword (ConvertTo-SecureString "##ammarTi2b" -AsPlainText -Force) -Enabled $true

   New-AdUser -Name "Nailis Saputri" -GivenName "Nailis" -Surname "Saputri" -SamAccountName "NailisSaputri" -UserPrincipalName "NailisSaputri@a3n4.com" -Path "OU=a3n4_Staff, DC=a3n4, DC=com" -AccountPassword (ConvertTo-SecureString "##nailisTi2b" -AsPlainText -Force) -Enabled $true

   New-ADUser -Name "Dhiyaul Atha" -GivenName "Dhiyaul" -Surname "Atha" -SamAccountName "DhiyaulAtha" -UserPrincipalName "DhiyaulAtha@a3n4.com" -Path "OU=a3n4_Staff,DC=a3n4,DC=com" -AccountPassword (ConvertTo-SecureString "##athaTi2b" -AsPlainText -Force) -Enabled $true

   New-ADUser -Name "Naiza Fitri" -GivenName "Naiza" -Surname "Fitri" -SamAccountName "NaizaFitri" -UserPrincipalName "NaizaFitri@a3n4.com" -Path "OU=a3n4_Staff,DC=a3n4,DC=com" -AccountPassword (ConvertTo-SecureString "##naizaTi2b" -AsPlainText -Force) -Enabled $true

   New-ADUser -Name "Nayla Mutia" -GivenName "Nayla" -Surname "Mutia" -SamAccountName "NaylaMutia" -UserPrincipalName "NaylaMutia@a3n4.com" -Path "OU=a3n4_Staff,DC=a3n4,DC=com" -AccountPassword (ConvertTo-SecureString "##naylaTi2b" -AsPlainText -Force) -Enabled $true

   New-ADUser -Name "Nikmal Wakil" -GivenName "Nikmal" -Surname "Wakil" -SamAccountName "NikmalWakil" -UserPrincipalName "NikmalWakil@a3n4.com" -Path "OU=a3n4_Staff,DC=a3n4,DC=com" -AccountPassword (ConvertTo-SecureString "##nikmalTi2b" -AsPlainText -Force) -Enabled $true

   New-ADUser -Name "Arini Safitri" -GivenName "Arini" -Surname "Safitri" -SamAccountName "AriniSafitri" -UserPrincipalName "AriniSafitri@a3n4.com" -Path "OU=a3n4_Staff,DC=a3n4,DC=com" -AccountPassword (ConvertTo-SecureString "##ariniTi2b" -AsPlainText -Force) -Enabled $true
   ```

4. **Add Users to Groups**

   ```PowerShell
   Add-AdGroupMember -Identity "HR_Staff" -Members "Ammar"
   Add-AdGroupMember -Identity "Finance_Staff" -Members "NailisSaputri"
   Add-AdGroupMember -Identity "Marketing_Staff" -Members "DhiyaulAtha"
   # Add other users to their respective groups as needed
   ```

### Steps - Remove

```PowerShell
Remove-ADUser -Identity "Nama_User"
Remove-ADGroup -Identity "Nama_Group"
Remove-ADOrganizationalUnit -Identity "OU=a3n4_Staff,DC=a3n4,DC=com"
```

### Steps - Verification

```PowerShell
Get-ADOrganizationalUnit -Filter *
Get-ADGroup -Filter * -SearchBase "OU=a3n4_Staff,DC=a3n4,DC=com"
Get-ADUser -Filter * -SearchBase "OU=a3n4_Staff,DC=a3n4,DC=com"
Get-ADGroupMember -Identity "All_Staff"
```
# IIS

## Function
### IIS (Internet Information Services) - Opening an Online Store

### Philosophy
After all infrastructure is ready, it is time to open an "online store" so the world can access the services we offer. Without IIS, the server is just an ordinary computer. With IIS, you can host websites, web applications, or online services accessible from anywhere.

### Steps - Create

```Powershell
Install-WindowsFeature -Name Web-Server -IncludeManagementTools

New-Item -Path "C:\inetpub\a3n4web" -ItemType Directory

New-Website -Name "a3n4web" -Port 80 -IpAddress "*" -HostHeader "a3n4.com" -PhysicalPath "C:\inetpub\a3n4web"

Copy-Item -Path "Z:\web_iis\*" -Destination "C:\inetpub\a3n4web"

Install-WindowsFeature Web-Static-Content

New-SelfSignedCertificate -DnsName "a3n4.com" -CertStoreLocation "cert:\LocalMachine\My"

Import-Module WebAdministration

New-WebBinding -Name "a3n4web" -Protocol https -Port 443 -IpAddress "*" -HostHeader "a3n4.com"

Get-Item "cert:\LocalMachine\My\<ThumbPrint>" | New-Item "IIS:SslBindings\0.0.0.0!443!a3n4.com"
```

### Steps - Remove

```Powershell
Remove-Website -Name "a3n4web"
Remove-Item "IIS:\SslBindings\0.0.0.0!443!a3n4.com" -ErrorAction SilentlyContinue
Remove-Item -Path "C:\inetpub\a3n4web" -Recurse -Force
Get-ChildItem Cert:\LocalMachine\My | Where-Object {$_.Subject -eq "CN=a3n4.com"} | Remove-Item
```

### Steps - Verification

#### Server
```Powershell
Get-WindowsFeature Web-Server
Get-Website -Name "a3n4web"
Get-WebBinding -Name "a3n4web"
```

#### Client
1. Log in to the Client.
2. Visit `http://a3n4.com` to access the IIS website.
3. Visit `https://a3n4.com` to test HTTPS.
4. A "Your Connection Isn't Private" warning will appear.
5. Click **Advanced** → **Continue to a3n4.com (unsafe)**.
6. The site will load successfully.

# IIS - Laravel

## Function
### Running a Laravel Application on IIS

### Philosophy
A Laravel application turns your IIS server into a full-stack web development platform. With PHP, MySQL, Composer, and Node.js, you can deploy modern web applications that integrate with your Active Directory domain.

### Steps - Prerequisites

Prepare the following installer files:

| File | Purpose |
|------|---------|
| `Composer-Setup.exe` | PHP dependency manager |
| `group_manage.sql` | Project database |
| `group-manage.zip` | Laravel project files |
| `mysql-8.0.46-winx64.zip` | MySQL database |
| `node-v22.17.1-x64.msi` | Node.js & npm |
| `php-8.4.12-Win32-vs17-x64.zip` | PHP interpreter |
| `rewrite_amd64_en-US.msi` | URL Rewrite module |
| `VC_redist.x64.exe` | Visual C++ Redistributable |
| `web.config` | IIS rewrite configuration |

### Steps - Install

#### 1. PHP Setup

```powershell
# Extract PHP
mkdir c:\php
Expand-Archive -Path "php.zip" -DestinationPath "c:\php"

# Add to PATH
$path = [System.Environment]::GetEnvironmentVariable("Path", "User")
[System.Environment]::SetEnvironmentVariable("Path", "$path;C:\php", "User")

# Verify
php -v
```

Edit `C:\php\php.ini` and uncomment these extensions:
```ini
extension=fileinfo
extension=mbstring
extension=openssl
extension=pdo_mysql
extension=mysqli
extension=curl
extension=zip
```

#### 2. MySQL Setup

```powershell
# Extract MySQL
mkdir c:\mysql
Expand-Archive -Path "mysql.zip" -DestinationPath "c:\mysql"

# Add to PATH
$path = [System.Environment]::GetEnvironmentVariable("Path", "User")
[System.Environment]::SetEnvironmentVariable("Path", "$path;C:\mysql\bin", "User")

# Verify
mysql --version
```

Create `C:\mysql\my.ini`:
```ini
[mysqld]
basedir=C:/MYSQL
datadir=C:/MYSQL/data
port=3306

[client]
port=3306
```

Initialize and install MySQL:
```powershell
mkdir c:\mysql\data
mysqld --initialize-insecure --basedir=C:\mysql --datadir=C:\mysql\data
mysqld --install MySQL --defaults-file=C:\mysql\my.ini
Start-Service MySQL
```

Import the project database:
```cmd
mysql -u root -e "create database group_manage"
mysql -u root group_manage < z:/iis-laravel/group_manage.sql
```

#### 3. Install Supporting Software

Run each installer:
- `VC_redist.x64.exe`
- `rewrite_amd64_en-US.msi`
- `Composer-Setup.exe` (ensure internet/NAT is active)
- `node-v22.17.1-x64.msi`

#### 4. IIS Configuration

```powershell
# Install IIS features
Install-WindowsFeature -Name Web-Server -IncludeManagementTools
Install-WindowsFeature -Name Web-CGI
Install-WindowsFeature -Name Web-Static-Content

# Configure FastCGI for PHP
Import-Module WebAdministration
Add-WebConfiguration -Filter /system.webServer/fastCgi -value @{fullpath='C:\php\php-cgi.exe'}
New-WebHandler -Name PHP-FastCGI -Path "*.php" -Verb "*" -Modules FastCgiModule -ScriptProcessor "C:\php\php-cgi.exe"
```

#### 5. Deploy Laravel Project

```powershell
# Extract project
Expand-Archive -Path "group-manage.zip" -DestinationPath "c:\inetpub\group-manage"

# Create DNS record
Add-DnsServerResourceRecordA -Name "group-manage" -ZoneName "a3n4.com" -IPv4Address "172.18.18.180"

# Create IIS website
Import-Module WebAdministration
New-WebSite -Name "group-manage" -Port 80 -HostHeader "group-manage.a3n4.com" -PhysicalPath "C:\inetpub\group-manage\public"

# Create and assign application pool
New-WebAppPool -Name "group-manage"
Set-ItemProperty -Path "IIS:\Sites\group-manage" -Name applicationPool -Value "group-manage"

# Set permissions
icacls "C:\inetpub\group-manage\storage" /grant "IIS AppPool\group-manage:(OI)(CI)F" /T
```

#### 6. Configure web.config

```powershell
cd C:\inetpub\group-manage
New-Item -Path "public\web.config" -ItemType "file"
notepad "public\web.config"
```

Paste the following content:
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

#### 7. Laravel Application Setup

Update the `.env` file with the correct MySQL password, then run:

```powershell
cd C:\inetpub\group-manage

composer install
php artisan key:generate
npm install
npm run build

# Remove the hot file
Remove-Item public\hot -ErrorAction SilentlyContinue

php artisan config:clear
php artisan cache:clear
php artisan route:clear
php artisan view:clear
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

#### 8. HTTPS Binding (Optional)

```powershell
# Create self-signed certificate
$cert = New-SelfSignedCertificate -DnsName "group-manage.a3n4.com" -CertStoreLocation "cert:\LocalMachine\My"

# Add HTTPS binding
Import-Module WebAdministration
New-WebBinding -Name "group-manage" -Protocol https -Port 443 -HostHeader "group-manage.a3n4.com"

# Associate certificate
New-Item -Path "IIS:\SslBindings\0.0.0.0!443!group-manage.a3n4.com" -Thumbprint $cert.Thumbprint -SslFlags 1
```

### Steps - Verification

#### Server
```powershell
Get-Service MySQL
Get-Service W3SVC
```

#### Client
1. Open a browser and visit `http://group-manage.a3n4.com`.
2. Register an account and create a project group.
3. Visit `https://group-manage.a3n4.com` and proceed past the certificate warning.

# Remote Desktop

## Function
### Remote Desktop - Virtual Office

### Philosophy
Sometimes you need to work from another location or manage the server without being physically in front of it. Remote Desktop allows you to control the server as if you were sitting right in front of it.

### Steps - Create

1. Open **SConfig**.
2. Select **7** (Remote Desktop).
3. Select **E** to enable Remote Desktop.
4. Select **1** (more secure).
5. Press Enter, then select **13** to restart.

### Steps - Remove

1. Open **SConfig**.
2. Select **7** (Remote Desktop).
3. Select **D** to disable Remote Desktop.
4. Press Enter, then select **13** to restart.

### Steps - Verification

#### Server
1. Open **SConfig**.
2. Select **7**.
3. Verify the status shows **Enabled**.
4. Press Enter, then select **13**.

#### Client
1. Log in to the Client.
2. Press `Win + R`, type `mstsc`, and press Enter.
3. In the Remote Desktop Connection window, enter the computer name: `a3n4.com`.
4. Click **Connect**.
5. Enter the user credentials when prompted.
6. Click **Yes** on the certificate warning (this is a self-built server, so it is safe).

# ADFS

## Function
### ADFS (Active Directory Federation Services) - External Security Gateway

### Philosophy
ADFS acts as a security gate that allows users from other organizations (e.g., business partners) to access resources in your network without creating new accounts for them. It enables secure collaboration between organizations.

### Steps - Create

```Powershell
Install-WindowsFeature ADFS-Federation -IncludeManagementTools

Add-DnsServerResourceRecordA -Name "adfs" -ZoneName "a3n4.com" -IPv4Address 172.18.18.180

New-SelfSignedCertificate -DnsName "adfs.a3n4.com" -CertStoreLocation "cert:\LocalMachine\My"

Add-KdsRootKey -EffectiveTime ((Get-Date).AddDays(1))

$pass = "2025" | ConvertTo-SecureString -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential ("A3N4\administrator", $pass)
Install-ADFSFarm -CertificateThumbprint "8A5421340313C07AA2017F76F55984BAC754C422" -FederationServiceName "adfs.a3n4.com" -FederationServiceDisplayName "A3N4 Federation Service" -ServiceAccountCredential $cred

Set-ADFSProperties -EnableIdpInitiatedSignOnPage $true

# Customize Theme
New-AdfsWebTheme -Name "CustomTheme" -SourceName "Default"
Set-AdfsWebTheme -TargetName "CustomTheme" -Illustration @{path="Z:\wall_a3n4.jpg"}
Set-AdfsWebTheme -TargetName "CustomTheme" -Logo @{path="Z:\logo_a3n4.png"}
Set-AdfsWebConfig -ActiveThemeName "CustomTheme"
```

### Steps - Remove

```Powershell
Uninstall-ADFSFarm
Get-ChildItem Cert:\LocalMachine\My | Where-Object {$_.Subject -like "*adfs*"} | Remove-Item
Remove-DnsServerResourceRecord -Name "adfs" -ZoneName "a3n4.com" -RRType A -Force
Remove-WindowsFeature ADFS-Federation -IncludeManagementTools
```

### Steps - Verification

#### Server
```Powershell
Get-WindowsFeature -Name ADFS-Federation
Get-Service ADFSSRV
Get-DnsServerResourceRecord -ZoneName "a3n4.com" -Name "adfs"
Get-ChildItem Cert:\LocalMachine\My\
Get-ADFSProperties | Select-Object HostName, EnableIdpInitiatedSignonPage, HttpPort, HttpsPort, CertificateDuration
```

#### Client
1. Log in to the Client.
2. Open a browser and visit: `https://adfs.a3n4.com/adfs/ls/IdpInitiatedSignOn.aspx`
3. A "Your connection is not private" warning will appear.
4. Click **Advanced** → **Continue to adfs.a3n4.com (unsafe)**.
5. The ADFS login page will load.
6. Click **Sign in** and enter your domain credentials.


# Ethernet 2 Unpropriety

## Function
### Ethernet 2 Unpropriety - Resolving Network Conflicts

### Philosophy
Sometimes you have more than one network adapter connected. To avoid confusion, you must set which adapter is primary and which is secondary.

### Steps - Configure

```Powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet 2" -ResetServerAddresses
Set-DnsClient -InterfaceAlias "Ethernet 2" -RegisterThisConnectionsAddress $false
Set-NetIPInterface -InterfaceAlias "Ethernet" -InterfaceMetric 10
Set-NetIPInterface -InterfaceAlias "Ethernet 2" -InterfaceMetric 50
```

# DFS
## Function
## Steps Install
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
Uninstall-WindowsFeature Fs-Dfs-NameSpace -IncludeManagementTools
```
## Verification Step
### Server
```Powershell
Get-WindowsFeature Fs-Dfs-NameSpace 
Get-DfsnRoot
Get-DfsnFolder -Path "\\a3n4.com\data_a3n4\*"
Get-SmbShare -Name DFSa3n4
Get-Acl "C:\Shared\AllStaff" | Format-List
Get-Acl "C:\Shared\HRStaff" | Format-List
Get-Acl "C:\Shared\FinanceStaff" | Format-List
Get-Acl "C:\Shared\MarketingStaff" | Format-List
```

### Client
1. Login Client
2. Klik kanan pada logo windows pilih run
3. Ketik \\a3n4.com\data_a3n4
4. Maka akan direct ke folder data_a3n4, yang menampilkan folder group yang dimasukin oleh user tsb.
5. Pada Folder Group User tsb buatlah sebuah file
6. Di Server masuk ke directory C:\Shared\<Folder_Group_user_sebelumnya>
7. Maka akan tampil file yang telah dibuat oleh user tsb. 


# FSRM
## Function
## Steps Install
```Powershell
Install-WindowsFeature -Name FS-Resource-Manager -IncludeManagementTools

Restart-Service SrmSvc

New-FsrmQuotaTemplate -Name "250MB" -Description "Maksimum 250MB" -Size 250MB
New-FsrmQuota -Path "C:\Shared\AllStaff" -Template "250MB"

# Memakai Template bawaan
New-FsrmQuota -Path "C:\Shared\FinanceStaff" -Template "100 MB Limit"
New-FsrmQuota -Path "C:\Shared\HRStaff" -Template "100 MB Limit"
New-FsrmQuota -Path "C:\Shared\MarketingStaff" -Template "100 MB Limit"

```

## Remove Step
```Powershell
Remove-FsrmQuota -Path "C:\Shared\FinanceStaff", "C:\Shared\HRStaff", "C:\Shared\MarketingStaff", "C:\Shared\AllStaff"

Remove-FsrmQuotaTemplate -Name "250MB", "100MB"

Uninstall-WindowsFeature -Name FS-Resource-Manager -IncludeManagementTools

```

## Verification Step
### Server
```Powershell
Get-WindowsFeature -Name FS-Resource-Manager

Get-FsrmQuota | Select-Object Path, Size, Template, Usage

Get-FSRMQuotaTemplate | Select-Object Name, Size
```

### Client
1. Login Client
2. Klik kanan pada logo windows pilih run
3. Ketik \\a3n4.com\data_a3n4
4. Maka akan direct ke folder data_a3n4, yang menampilkan folder group yang dimasukin oleh user tsb.
5. Pada Folder Group User tsb tambahkan beberapa file dan folder
6. Di Server masuk klik command `Get-FsrmQuota`
7. Maka akan tampil berapa byte yang telah dipakai oleh folder tsb

# NTP
## Function
## Steps Install
```Powershell
w32tm /Config /ManualPeerList:"SRV01,0x1" /SyncFromFlags:Manual /Reliable:YES /Update

Start-Service w32time

w32tm /Config /ManualPeerList:"Time.Windows.Com,0x1" /SyncFromFlags:Manual /Reliable:YES /Update

w32tm /ReSync
```

## Remove Step
```Powershell

w32tm /unregister
w32tm /register
Restart-Service w32time
w32tm /resync
```

## Verification Step
### Server
```Powershell
w32tm /Query /Status
w32tm /Query /Source
```

### Client
1. Login Client
2. Pada Kolom Search cari Date & Time Setting
3. Pada Bagian Synchronize your clock
4. Maka tampak tombol `Sync Now` tidak bisa ditekan

# FTP
## Function
## Step Install
```Powershell
Install-WindowsFeature -Name Web-FTP-Server, Web-FTP-Service, Web-Mgmt-Service, Web-Server

New-Item -Path "C:\FTPRoot" -ItemType Directory

Import-Module WebAdministration

New-WebFtpSite -Name "A3N4FTP" -PhysicalPath "C:\FTPRoot" -port 21 -IPAddress "*"

Set-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST' -filter "system.applicationHost/sites/site[@name='A3N4FTP']/ftpServer/security/ssl" -name controlChannelPolicy -value "SslAllow"
Set-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST' -filter "system.applicationHost/sites/site[@name='A3N4FTP']/ftpServer/security/ssl" -name dataChannelPolicy -value "SslAllow"

Set-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST' -filter "system.applicationHost/sites/site[@name='A3N4FTP']/ftpServer/security/authentication/basicAuthentication" -name enabled -value true

Add-WebConfigurationProperty -PSPath 'MACHINE/WEBROOT/APPHOST' -Filter "system.applicationHost/sites/site[@name='A3N4FTP']/ftpServer/security/authorization" -Name "." -Value @{AccessType="Allow"; Users="*"; Permissions="Read, Write"}

$acl = Get-Acl "C:\FTPRoot"
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule("Everyone","Modify","Allow")
$acl.SetAccessRule($rule)
Set-Acl "C:\FTPRoot" $acl
```

## Steps Remove
```Powershell
Remove-WebSite -Name "A3N4FTP"

Clear-WebConfiguration -pspath 'MACHINE/WEBROOT/APPHOST' -filter "system.applicationHost/sites/site[@name='A3N4FTP']"

iisreset
```

## Steps Verification
### Server
```Powershell
Get-WindowsFeature -Name Web-FTP-Server, Web-FTP-Service, Web-Mgmt-Service, Web-Server

Get-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST' -filter "system.applicationHost/sites/site[@name='A3N4FTP']/ftpServer/security/ssl" -name *

Get-WebConfiguration -PSPath 'MACHINE/WEBROOT/APPHOST' -Filter "system.applicationHost/sites/site[@name='A3N4FTP']"

Get-Website -Name A3N4FTP
```

### Client
1. Login Client.
2. Buka file explorer, pada url.
3. masukkan `ftp://192.168.56.10` dan enter.
4. Maka muncul Window Masukkan Nama user yang sedang login dan passwordnya.
5. Buat Folder Baru pada directory tsb.
6. Pada Server masukkan command `ls C:\FTPRoot`.
7. Maka akan tampil file yang telah dibuat oleh user tsb. 


# Mail Server
## Function
## Steps Install
1. Pada server jalankan .exe mailEnable yang telah disiapkan
2. Jika Belum Copy file dari harddisk atau shared folder ke dir "C:\"
3. Lalu jalankan file .exe dengan "namafile.exe"
4. Ikuti langkah instalasi dengan klik next 
5. Untuk jendela warning Klik Ok
6. Pada Select Component centang semua, Klik Next
7. Pada Select Destination Location, klik Next
8. Pada Enter The name of the program... pilih Mail Enable, Klik Next
9. Pada Enter Post Office dan Password Masukkan domain.com dan password domain
10. Pada Masukkan SMTP Masukkan Nama Domain, DNS Host, dan SMTP Port, Klik Next, Lalu Finish

```Powershell
Add-DnsServerResourceRecordA -Name "mail" -ZoneName "a3n4.com" -IPv4Address "192.168.56.10" -CreatePtr 
Add-DnsServerResourceRecordCName -Name "autodiscover" -HostNameAlias "mail.a3n4.com" -ZoneName "a3n4.com"
Add-DnsServerResourceRecord -ZoneName "a3n4.com" -Name "@" -Txt -DescriptiveText "v=spf1 ip4::192.168.56.10 a mx ptr include:mail.a3n4.com -all"
Add-DnsServerResourceRecordA -Name "webmail" -ZoneName "a3n4.com" -IPv4Address "192.168.56.10" -CreatePtr

Restart-Service DNS
Import-Module WebAdministration

New-WebBinding -name "MailEnable WebMail" -IPAddress "192.168.56.10" -port 80 -HostHeader "mail.a3n4.com" -Protocol "http"
New-WebBinding -name "MailEnable WebAdmin" -IPAddress "192.168.56.10" -port 80 -HostHeader "webmail.a3n4.com" -Protocol "http" 

IISReset

cd 'C:\Program Files (x86)\mail enable\postoffices\a3n4.com\mailroot'

Add-PssNapin mailenable.provision.command
New-MailEnableMailBox -Domain "a3n4.com" -Mailboxes "AmmarShiddiq" -Password "#@amram3" -Right "ADMIN"
New-MailEnableMailBox -domain "a3n4.com" -mailboxes "AriniSaputri" -password "#@rar1n" -rights "USER"

cd ..\..\..\bin

MEInstaller.exe SetDefaultMailServer -MailServer "mail.a3n4.com"
MEInstaller.exe setpopalternateport -port 995 -requiressl
MEInstaller.exe setimapalternateport -port 993 -requiressl 
```


## Steps Remove
```Powershell
#Uninstall Mail Server

Remove-DnsServerResourceRecord -ZoneName "a3n4.com" -RRType "A" -Name "mail" -Force
Remove-DnsServerResourceRecord -ZoneName "a3n4.com" -RRType "A" -Name "webmail" -Force
Remove-DnsServerResourceRecord -ZoneName "a3n4.com" -RRType "CNAME" -Name "autodiscover" -Force
Remove-DnsServerResourceRecord -ZoneName "a3n4.com" -RRType "TXT" -Name "@" -Force

Remove-WebBinding -Name "MailEnable WebMail" -Port 80 -HostHeader "mail.a3n4.com"
Remove-WebBinding -Name "MailEnable WebAdmin" -Port 80 -HostHeader "webmail.a3n4.com"

Remove-Item "C:\Program Files (x86)\Mail Enable\"

# Cari Service yang masih tinggal -> Get-Service -DisplayName *mail*
sc delete "NamaService"
```

## Steps Verification
### Server
```Powershell
Get-DnsServerResourceRecord -ZoneName "a3n4.com" -Name "mail"
Get-DnsServerResourceRecord -ZoneName "a3n4.com" -Name "webmail"
Get-DnsServerResourceRecord -ZoneName "a3n4.com" -Name "autodiscover"
Get-DnsServerResourceRecord -ZoneName "a3n4.com" -Name "@" -RRType TXT
Get-WebBinding -Name "MailEnable WebMail"
Get-WebBinding -Name "MailEnable WebAdmin"
Get-Service -DisplayName *mail*

```


### Client
1. Login Client
2. Buka Browser lalu ketik http://mail.a3n4.com
3. Maka akan tampil halaman login mail enable
4. Masukkan username admin dan password lalu klik login
5. Klik New Email Message
6. Masukkan email tujuan lalu klik send
7. Logout Dari admin
8. Masukkan username user dan password lalu klik login
9. Klik Inbox
10. Maka akan tampil email yang telah dikirim oleh admin


# VPN
## Function
## Step Install
```Powershell
Install-WindowsFeature -Name RemoteAccess -IncludeManagementTools

Install-WindowsFeature -Name RSAT-RemoteAccess, DirectAccess-VPN, Routing, RSAT-AD-Powershell

# Restart-Computer

Install-RemoteAccess -VpnType VPN

Set-ADUser -Identity "AmmarShiddiq" -Replace @{msNPAllowDialin = $true}


```

## Steps Remove
```Powershell
Uninstall-WindowsFeature -Name DirectAccess-VPN, Routing, RSAT-RemoteAccess, RSAT-AD-Powershell

Uninstall-WindowsFeature -Name RemoteAccess -IncludeManagementTools

Uninstall-RemoteAccess -VpnType VPN

Set-ADUser -Identity "AmmarShiddiq" -Replace @{msNPAllowDialin = $false}
```

## Steps Verification
### Server
```Powershell
Get-WindowsFeature -Name RemoteAccess, RSAT-RemoteAccess, DirectAccess-VPN, Routing, RSAT-AD-Powershell

Get-Service RemoteAccess

Get-RemoteAccess | Select-Object VpnStatus

Get-ADUser -Identity "AmmarShiddiq" -Properties msNPAllowDialin | Select-Object Name, msNPAllowDialin
Get-ADUser -Filter * -Properties msNPAllowDialin | Select-Object Name, msNPAllowDialin
```

### Client
1. Login Client
2. Klik kanan pada logo windows pilih run
3. Ketik 'control.exe /name Microsoft.NetworkAndSharingCenter'
4. Pada Window Network and Sharing Center Klik Set up a new connection or network
5. Pilih Connect to a workplace lalu klik Next
6. Pilih Use my Internet connection (VPN) lalu ill set up an internet connection later
7. Pada Internet Address masukkan ip address server dan destination name
8. Klik Create
9. Klik logo internet bagian bawah
10. Klik nama VPN yang telah dibuat lalu klik Connect
11. Masukkan username dan password lalu klik Connect
12. Maka akan terhubung ke VPN Server 