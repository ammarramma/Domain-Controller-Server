# Install Ubuntu
# -->-->-->

# Config IP
```bash
sudo apt update && sudo apt upgrade
sudo nano /etc/netplan/00-installer-config.yaml
```
## Dan isi ->:
```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: false
      addresses:
        - 172.18.18.180/24
      nameservers:
        addresses:
          - 172.18.18.180
```
```bash
sudo chmod 666 /etc/netplan/00-installer-config.yaml
sudo netplan apply
```

## Verifikasi ->:
```bash
ip a
```

# ADDC
```bash
sudo apt update
sudo apt install samba krb5-user winbind smbclient dnsutils -y
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.bak

sudo samba-tool domain provision
```
## Then ->:
```choose
Realm: AMMAR.COM
Domain: AMMAR
Server Role: dc
DNS backend: SAMBA_INTERNAL
DNS forwarder IP: 8.8.8.8
Administrator password: #aMMar#
```
```bash
# Copy Kerberos Config
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf

# Stop Service Lama
sudo systemctl stop smbd nmbd winbind
sudo systemctl disable smbd nmbd winbind

# Enable Samba AD DC
sudo systemctl unmask samba-ad-dc
sudo systemctl enable samba-ad-dc
sudo systemctl start samba-ad-dc
```

## Verifikasi ->:
```bash
sudo samba-tool domain info 127.0.0.1

systemctl status samba-ad-dc

kinit administrator

klist
```

## Remove ->:
```
sudo systemctl stop samba-ad-dc
sudo systemctl disable samba-ad-dc

sudo apt remove samba krb5-user winbind smbclient -y
sudo apt autoremove -y

sudo rm -rf /etc/samba
sudo rm -rf /var/lib/samba
```

# DHCP
```bash
sudo apt update
sudo apt install isc-dhcp

sudo nano /etc/default/isc-dhcp-server
```
## And Change ->:
```bash
INTERFACESv4="" 
to ->:
INTERFACESv4="enp0s8"
```

```bash
sudo nano /etc/dhcp/dhcpd.conf
```
## And ADD ->:
```bash
authoritative;

subnet 172.18.18.0 netmask 255.255.255.0 {
    range 172.18.18.181 172.18.18.200;

    option routers 172.18.18.1;
    option domain-name-servers 172.18.18.180;
    option domain-name "ammar.com";

    default-lease-time 31536000;
    max-lease-time 31536000;
}
```
```bash
sudo systemctl restart isc-dhcp-server
sudo systemctl enable isc-dhcp-server
```

## Verifikasi ->:
```bash
systemctl status isc-dhcp-server
cat /var/lib/dhcp/dhcpd.leases
sudo ss -tulpn | grep dhcp
journalctl -u isc-dhcp-server
```

## Remove ->:
```bash
sudo systemctl stop isc-dhcp-server
sudo systemctl disable isc-dhcp-server

sudo apt remove isc-dhcp-server -y
sudo apt autoremove -y

sudo rm /etc/dhcp/dhcpd.conf
```

# DNS
```bash
-- Forward
sudo samba-tool dns add 127.0.0.1 ammar.com www A 172.18.18.180 -U administrator

-- Reverse
sudo samba-tool dns zonecreate 127.0.0.1 18.18.172.in-addr.arpa -U administrator

-- PTR
sudo samba-tool dns add 127.0.0.1 18.18.172.in-addr.arpa 180 PTR www.ammar.com -U administrator
```

## Verifikasi ->:
```bash
sudo samba-tool dns zonelist 127.0.0.1 -U administrator
sudo samba-tool dns query 127.0.0.1 ammar.com @ ALL -U administrator
cat /etc/resolv.conf
ping www.ammar.com
nslookup www.ammar.com
nslookup 172.18.18.180
```

## Remove ->:
```bash
sudo samba-tool dns delete 127.0.0.1 ammar.com www A 172.18.18.180 -U administrator
sudo samba-tool dns delete 127.0.0.1 18.18.172.in-addr.arpa 180 PTR www.ammar.com -U administrator
sudo samba-tool dns zonedelete 127.0.0.1 18.18.172.in-addr.arpa -U administrator
```

# Join Domain
-- like windows create step

## Verifikasi ->:
```pwsh
systeminfo | select-string domain
```


# User & group
```bash
# Organization
sudo samba-tool ou create "OU=ammar_staff,DC=ammar,DC=com"
# Group
sudo samba-tool group add All-Staff
sudo samba-tool group add Marketing
sudo samba-tool group add Finance
sudo samba-tool group add HR
# User
sudo samba-tool user create ammar Passwd123
sudo samba-tool user create arini Passwd123
sudo samba-tool user create nayla Passwd123
sudo samba-tool user create nikmal Passwd123
sudo samba-tool user create nailis Passwd123
sudo samba-tool user create naiza Passwd123
sudo samba-tool user create atha Passwd123
# add user to Organization
sudo samba-tool user show ammar
sudo samba-tool user move ammar "OU=ammar_staff,DC=ammar,DC=com"
for u in ammar arini naila nailis naiza nikmal atha; do sudo samba-tool user move "$u" "OU=ammar_staff,DC=ammar,DC=com"; done
# Members
sudo samba-tool group addmembers HR ammar
for u in ammar arini ; do sudo samba-tool group addmembers Marketing "$u"; done
for u in naila nailis; do sudo samba-tool group addmembers HR "$u"; done
for u in naiza nikmal atha; do sudo samba-tool group addmembers Finance "$u"; done
```

## Verification ->:
```bash
sudo samba-tool ou list
sudo samba-tool group list
sudo samba-tool user list
for g in HR Marketing Finance All-Staff; do  echo "$g"; sudo samba-tool group listmembers "$g"; echo ""; done
```

## Remove ->:
```bash
sudo samba-tool ou delete "OU=ammar_staff, DC=ammar, DC=com"
sudo samba-tool group delete Finance
sudo samba-tool user delete ammar
```

# Nginx
```bash
sudo apt update
sudo apt install nginx -y

sudo mkdir -p /var/www/ammarweb
sudo nano /var/www/ammarweb/index.html
# isi dengan yang telah disiapkan

sudo nano /etc/nginx/sites-available/ammarweb
```
Configure Virtual Host, Setelah itu isi dengan:
```ammarweb
server {
    listen 80;
    server_name ammar.com www.ammar.com;

    root /var/www/ammarweb;
    index index.html;
}
```

```bash
# Enable website
sudo ln -s /etc/nginx/sites-available/ammarweb \
/etc/nginx/sites-enabled/

#Verify Configuration
sudo nginx -t

# restart nginx
sudo systemctl restart nginx
sudo systemctl enable nginx

#create self-signed-certificate
sudo openssl req -x509 -nodes -days 365 \
-newkey rsa:2048 \
-keyout /etc/ssl/private/ammar.key \
-out /etc/ssl/certs/ammar.crt

# Configure HTTPS
sudo nano /etc/nginx/sites-available/ammarweb
```
```ammarweb
server {
    listen 443 ssl;

    server_name ammar.com www.ammar.com;

    ssl_certificate /etc/ssl/certs/ammar.crt;
    ssl_certificate_key /etc/ssl/private/ammar.key;

    root /var/www/ammarweb;
    index index.html;
}
```
```bash
#Reload nginx
sudo systemctl reload nginx
```

## Verification ->:
```bash
systemctl status nginx
sudo nginx -t
ss -tulpn | grep nginx
```
Untuk Client buka browser dan browse:
`http://www.ammar.com` -> http
`https://www.ammar.com` -> https

## Remove ->:
```bash
sudo rm /etc/nginx/sites-enabled/ammarweb

sudo rm /etc/nginx/sites-available/ammarweb

sudo rm -rf /var/www/ammarweb

sudo systemctl reload nginx

#optional 
sudo apt remove nginx -y
sudo apt autoremove -y
```

## SSH
Menggunakan SSH melalui terminal client 
yang perlu disiapkan:
```bash
# Install OpenSSH Server
sudo apt update
sudo apt install openssh-server -y

# Enable SSH
sudo systemctl enable ssh
sudo systemctl start ssh
```

## Verification ->:
```bash
systemctl status ssh
```

Dari client buka terminal dan masukkan:
```powershell
ssh ammar@172.18.18.180
```

# Keycloak
```bash
# Install Docker
sudo apt update
sudo apt install docker.io -y

# Run Keycloak
sudo docker run -d \
--name keycloak \
-p 8080:8080 \
-e KEYCLOAK_ADMIN=administrator \
-e KEYCLOAK_ADMIN_PASSWORD=#aMMar# \
 

# Add DNS Record
```

# DFS
1. Buat folder fisik storage
```bash
sudo mkdir -p /srv/storage/allstaff
sudo mkdir -p /srv/storage/hr
sudo mkdir -p /srv/storage/finance
sudo mkdir -p /srv/storage/marketing
```

2. Buat folder namespace
```bash
sudo mkdir -p /srv/dfsroot
```

3. Buat symlink ke storage
```bash
sudo ln -s /srv/storage/allstaff /srv/dfsroot/All-Staff
sudo ln -s /srv/storage/hr /srv/dfsroot/HR
sudo ln -s /srv/storage/finance /srv/dfsroot/Finance
sudo ln -s /srv/storage/marketing /srv/dfsroot/Marketing
```

Strukturnya jadi:
```struktur
/srv/dfsroot
├── All-Staff -> /srv/storage/allstaff
├── HR -> /srv/storage/hr
├── Finance -> /srv/storage/finance
└── Marketing -> /srv/storage/marketing
```

3. 5. Extra 
```bash
Backup dulu:
sudo cp /etc/nsswitch.conf /etc/nsswitch.conf.bak

# Edit:
sudo nano /etc/nsswitch.conf

# Ubah:
passwd:         files systemd
group:          files systemd

# menjadi:
passwd:         files systemd winbind
group:          files systemd winbind
passwd:         files systemd
group:          files systemd
```
3. 6. Install nss
```bash
sudo apt install libnss-winbind libpam-winbind -y
```

4. Set group owner
Cari dulu GID:
```bash
getent group all-staff
getent group finance
getent group marketing
getent group all-staff
```

Kalau berhasil:
```bash
sudo chgrp "All-Staff" /srv/storage/allstaff
sudo chgrp "HR" /srv/storage/hr
sudo chgrp "Finance" /srv/storage/finance
sudo chgrp "Marketing" /srv/storage/marketing
```

5. Set permission
```bash
sudo chmod 2770 /srv/storage/allstaff
sudo chmod 2770 /srv/storage/hr
sudo chmod 2770 /srv/storage/finance
sudo chmod 2770 /srv/storage/marketing
```

6. Edit Samba config
```bash
sudo nano /etc/samba/smb.conf
```

Tambahkan ->:
```conf
[data]
    path = /srv/dfsroot
    read only = no
    browseable = yes

    follow symlinks = yes
    wide links = yes
    unix extensions = no

    inherit permissions = yes
    hide unreadable = yes

    valid users = @"AMMAR\All-Staff" @"AMMAR\HR" @"AMMAR\Finance" @"AMMAR\Marketing"
```

7. Restart Samba
```bash
sudo testparm
sudo systemctl restart samba-ad-dc
```

## Verfikasi ->:
1. login client
2. jalankan windows + r
3. masukkan \\ammar-server\data

# Quota
1. Menginstall 
```bash
sudo apt install quota -y
```

2. Konfigurasi file 
```bash
sudo nano /etc/fstab
```
```bash
# Cari:
/dev/sda2

# misalnya:
UUID=xxxxx / ext4 defaults 0 1

# ubah menjadi:
UUID=xxxxx / ext4 defaults,usrquota,grpquota 0 1
```

3. Reboot Server
```bash
sudo reboot
```

4. Generate Database
```bash
sudo quotaoff / 
sudo quotacheck -cugm /
sudo quotaon /
```
Penjelasan:
-c create database
-u user quota
-g group quota
-m mounted filesystem
-f force

5. Check Hasil DB
```bash
ls -l /aquota*
sudo repquota /
```
Hasil1 -> list file
Hasil2 -> tabel disk

6. Membuat Template
```bash
QUOTA_100MB=102400
QUOTA_250MB=256000
```
7. Menerapkan Quota Limit
```bash
sudo setquota -g "AMMAR\\hr" $QUOTA_100MB $QUOTA_100MB 0 0 /
sudo setquota -g "AMMAR\\finance" $QUOTA_100MB $QUOTA_100MB 0 0 /
sudo setquota -g "AMMAR\\marketing" $QUOTA_100MB $QUOTA_100MB 0 0 /
sudo setquota -g "AMMAR\\all-staff" $QUOTA_250MB $QUOTA_250MB 0 0 /
```

# FTP
## Konfigurasi ->:
1. Install FTP Server
```bash
sudo apt update
sudo apt install vsftpd -y
```

2. Backup Config
```bash
sudo cp /etc/vsftpd.conf /etc/vsftpd.conf.bak
```

3. Buat FTP Root
```bash
sudo mkdir -p /srv/ftp
```

4. Set Permission FTP Root
Jika anda ingin agar semua user bisa upload/download, gunakan group All-Staff, jika tidak anda dapat mengganti dengan user spesifik.

Pastikan group terbaca:
```bash
getent group "AMMAR\\all-staff"
```

Set owner:
```bash
sudo chgrp "AMMAR\\all-staff" /srv/ftp
sudo chmod 2770 /srv/ftp
```

5. Konfigurasi vsftpd
Edit:
```bash
sudo nano /etc/vsftpd.conf
```

Isi atau sesuaikan:
```bash
listen=YES
listen_ipv6=NO

anonymous_enable=NO
local_enable=YES
write_enable=YES

local_umask=002

chroot_local_user=YES
allow_writeable_chroot=YES

user_sub_token=$USER
local_root=/srv/ftp

pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40100

ssl_enable=NO
```
Selanjutnya masuk ke dalam file /etc/samba/smb.conf dan tambahkan:
```conf
template homedir = /srv/ftp
```

6. Restart Service
```bash

restart:
```bash
sudo systemctl restart samba-ad-dc
sudo systemctl restart vsftpd
sudo systemctl enable vsftpd
```

## Plus Plus
```bash
echo /bin/false | sudo tee -a /etc/shells


```

## Verify ->:
Verifikasi Status Service
```bash
systemctl status vsftpd
```

**Cek port:**
```bash
sudo ss -tulpn | grep :21
```
Harus muncul vsftpd.

## Steps Verification Client ->:
Dari Windows Client powershell:
```powershell
ftp 172.18.18.180
```

Login pakai user Linux lokal, misalnya:
ammar

atau user domain kalau PAM/winbind login sudah aktif.

Command FTP:
```ftp
ls
put file.txt
get file.txt
bye
```
```ftp
ftp ammmar@ammar.com
# put password

lcd /dirtoitskey
put user.ovpn
bye
```
**Catatan:** 
pastikan anda telah Menginstall 
```bash
sudo apt install libpam-winbind -y
```
jika anda ingin menggunakan user domain untuk mengakses ftp

## Steps Remove
```bash
sudo systemctl stop vsftpd
sudo systemctl disable vsftpd

sudo apt remove vsftpd -y
sudo apt autoremove -y

sudo rm -rf /srv/ftp
sudo rm -f /etc/vsftpd.conf
sudo mv /etc/vsftpd.conf.bak /etc/vsftpd.conf 2>/dev/null
```

# Strong Swan
1. Instalasi strongswan
```bash
sudo apt install strongswan xl2tpd ppp lsof net-tools -y
sudo nano /etc/ipsec.conf
```
add
```conf
config setup
        charondebug="ike 2, knl 2, cfg 2"

conn L2TP-PSK
        keyexchange=ikev1
        authby=secret
        type=transport
        left=172.18.18.180
        leftprotoport=17/1701
        right=%any
        rightprotoport=17/%any
        ike=aes128-sha1-modp1024
        esp=aes128-sha1
        auto=add
```
```bash
sudo nano /etc/ipsec.secrets
```
add
```conf
172.18.18.180 %any : PSK "ammarvpn123"
```

2. Konfigurasi L2TP 
```bash
sudo nano /etc/xl2tpd/xl2tpd.conf
```
add
```conf
[global]
ipsec saref = yes
listen-addr = 172.18.18.180

[lns default]
ip range = 172.18.18.181-172.18.18.200 
local ip = 172.18.18.180
refuse pap = yes 
refuse chap = yes 
require authentication = yes
name = ammarVpn
ppp debug = yes
pppoptfile = /etc/ppp/options.xl2tpd
length bit = yes
```

3. Konfigurasi PPP(Autentikasi User VPN)
```bash
sudo nano /etc/ppp/options.xl2tpd
```
isi
```conf
require-mschap-v2
ms-dns 172.18.18.180

asyncmap 0
auth
crtscts
lock
hide-password
modem
debug

name l2tpd

proxyarp

lcp-echo-interval 30
lcp-echo-failure 4
```

```bash
sudo nano /etc/ppp/chap-secrets
```
add
```conf
ammar l2tp userpass123 *
```

4. Aktifkan IP Forwarding & NAT
```bash
sudo nano /etc/sysctl.conf
```
Delete #
```conf
#from
#net.ipv4.ip_forward=1
#to 
net.ipv4.ip_forward=1
```
```bash
sudo sysctl -p

sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUARADE
sudo apt install iptables-persistent -y 
sudo netfilter-persistent save 

sudo systemctl restart strongswan-starter
sudo systemctl restart xl2tpd
```

# Mail Server 
## Konfigurasi ->:
Install
```bash
sudo apt install nginx php-fpm php-mysql php-intl php-mbstring php-xml php-curl php-zip -y
sudo apt install mariadb-server postfix dovecot-imapd dovecot-pop3d roundcube roundcube-mysql
```
buka file:
```bash
sudo nano /etc/nginx/sites-available/mail
```
Abistu isi
```conf
server {
    listen 80;
    server_name mail.ammar.com;

    root /usr/share/roundcube;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }

    location ~ ^/(config|temp|logs)/ {
        deny all;
    }
}
```
Konfigurasi website:
```bash
sudo ln -s /etc/nginx/sites-available/mail /etc/nginx/sites-enabled/mail
```
Restart
```bash
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl restart php8.3-fpm
```

sudo sysctl -p
sudo sysctl -psudo sysctl -psudo sysctl -p

## Keycloak V2
```bash
sudo samba-tool dns add 127.0.0.1 ammar.com keycloak A 172.18.18.180 -U administrator
sudo apt install default-jdk -y
cd /opt
sudo wget https://github.com/keycloak/keycloak/releases/download/24.0.3/keycloak-24.0.3.tar.gz 
sudo tar -xvzf keycloak-24.0.3.tar.gz 
sudo mv keycloak-24.0.3 keycloak
sudo ./bin/kc.sh build
KEYCLOAK_ADMIN=ammar
KEYCLOAK_ADMIN_PASSWORD=Passwd2026 
sudo ./bin/kc.sh start-dev --http-host=0.0.0.0 --hostname=keycloak.ammar.com
# ctrl+z 
bg 
```
