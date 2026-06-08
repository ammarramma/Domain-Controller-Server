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

# ADDS
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
systeminfo | serect-string domain
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

# nginx
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
## Untuk Client buka browser dan browse:
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

## Remote Desktop
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

# ADFS
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
-e KEYCLOAK_HOSTNAME=keycloak.ammar.com \
quay.io/keycloak/keycloak:latest start-dev

# Add DNS Record
sudo samba-tool dns add 127.0.0.1 ammar.com keycloak A 172.18.18.180 -U administrator
```

## Verification ->:
```bash
# Check Container
sudo docker ps

# Check Service
curl http://localhost:8080

# Check DNS
nslookup keycloak.ammar.com
```

## Client ->:
1. Login Client
2. Buka browser
3. Akses:
  http://keycloak.ammar.com:8080
4. Login menggunakan:
5. Username: administrator
6. Password: #aMMar#
Dashboard Keycloak berhasil ditampilkan

## Remove ->:
```bash
# Stop container:
sudo docker stop keycloak

# Remove container:
sudo docker rm keycloak

# Remove DNS record:
sudo samba-tool dns delete 127.0.0.1 ammar.com keycloak A 172.18.18.180 -U administrator
```
