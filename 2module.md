# Настроить доменный контроллер Samba на машине BR-SRV
## HQ-SRV
```bash
echo "server=/au-team.irpo/192.168.3.10" >> /etc/dnsmasq.conf
systemctl restart dnsmasq
```
## BR-SRV
```bash
if ! grep -q '^nameserver 8\.8\.8\.8$' /etc/resolv.conf; then
    echo 'nameserver 8.8.8.8' | sudo tee -a /etc/resolv.conf
fi
apt-get update && apt-get install wget dos2unix task-samba-dc -y
sleep 3
echo nameserver 192.168.1.10 >> /etc/resolv.conf
sleep 2
echo 192.168.3.10 br-srv.au-team.irpo >> /etc/hosts
rm -rf /etc/samba/smb.conf
samba-tool domain provision --realm=AU-TEAM.IRPO --domain=AU-TEAM --adminpass=P@ssw0rd --dns-backend=SAMBA_INTERNAL --server-role=dc --option='dns forwarder=192.168.1.10'
mv -f /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl enable --now samba.service
samba-tool user add hquser1 P@ssw0rd
samba-tool user add hquser2 P@ssw0rd
samba-tool user add hquser3 P@ssw0rd
samba-tool user add hquser4 P@ssw0rd
samba-tool user add hquser5 P@ssw0rd
samba-tool group add hq
samba-tool group addmembers hq hquser1,hquser2,hquser3,hquser4,hquser5
wget https://raw.githubusercontent.com/sudo-project/sudo/main/docs/schema.ActiveDirectory
dos2unix schema.ActiveDirectory
sed -i 's/DC=X/DC=au-team,DC=irpo/g' schema.ActiveDirectory
head -$(grep -B1 -n '^dn:$' schema.ActiveDirectory | head -1 | grep -oP '\d+') schema.ActiveDirectory > first.ldif
tail +$(grep -B1 -n '^dn:$' schema.ActiveDirectory | head -1 | grep -oP '\d+') schema.ActiveDirectory | sed '/^-/d' > second.ldif
ldbadd -H /var/lib/samba/private/sam.ldb first.ldif --option="dsdb:schema update allowed"=true
ldbmodify -v -H /var/lib/samba/private/sam.ldb second.ldif --option="dsdb:schema update allowed"=true
samba-tool ou add 'ou=sudoers'
cat << EOF > sudoRole-object.ldif
dn: CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo
changetype: add
objectClass: top
objectClass: sudoRole
cn: prava_hq
name: prava_hq
sudoUser: %hq
sudoHost: ALL
sudoCommand: /bin/grep
sudoCommand: /bin/cat
sudoCommand: /usr/bin/id
sudoOption: !authenticate
EOF
ldbadd -H /var/lib/samba/private/sam.ldb sudoRole-object.ldif
echo -e "dn: CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo\nchangetype: modify\nreplace: nTSecurityDescriptor" > ntGen.ldif
ldbsearch  -H /var/lib/samba/private/sam.ldb -s base -b 'CN=prava_hq,OU=sudoers,DC=au-team,DC=irpo' 'nTSecurityDescriptor' | sed -n '/^#/d;s/O:DAG:DAD:AI/O:DAG:DAD:AI\(A\;\;RPLCRC\;\;\;AU\)\(A\;\;RPWPCRCCDCLCLORCWOWDSDDTSW\;\;\;SY\)/;3,$p' | sed ':a;N;$!ba;s/\n\s//g' | sed -e 's/.\{78\}/&\n /g' >> ntGen.ldif
ldbmodify -v -H /var/lib/samba/private/sam.ldb ntGen.ldif
```
## HQ-CLI
```bash
sed -i 's/BOOTPROTO=static/BOOTPROTO=dhcp/' /etc/net/ifaces/ens20/options
systemctl restart network
apt-get update && apt-get install bind-utils -y
system-auth write ad AU-TEAM.IRPO cli AU-TEAM 'administrator' 'P@ssw0rd'
reboot
```
```bash
apt-get install sudo libsss_sudo -y
control sudo public
sed -i '19 a\
sudo_provider = ad' /etc/sssd/sssd.conf
sed -i 's/services = nss, pam/services = nss, pam, sudo/' /etc/sssd/sssd.conf
sed -i '28 a\
sudoers: files sss' /etc/nsswitch.conf
rm -rf /var/lib/sss/db/*
sss_cache -E
systemctl restart sssd
```

---

# Сконфигурировать файловое хранилище
## HQ-SRV
```bash
apt-get update && apt-get install fdisk mdadm -y
mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sd{b,c}
mdadm --detail -scan --verbose > /etc/mdadm.conf
echo -e "n\np\n1\n2048\n4186111\nw" | fdisk /dev/md0
mkfs.ext4 /dev/md0p1
echo "/dev/md0p1  /raid  ext4  defaults  0  0" >> /etc/fstab
mkdir /raid
mount -a
mount -v
```

---

# Настроить сетевую файловую систему
## HQ-SRV
```bash
apt-get update && apt-get install nfs-server -y
mkdir /raid/nfs
chown 99:99 /raid/nfs
chmod 777 /raid/nfs
echo "/raid/nfs  192.168.2.10/28(rw,sync,no_subtree_check)" >> /etc/exports
exportfs -a
exportfs -v
systemctl enable nfs && systemctl restart nfs
```
## HQ-CLI
```bash
apt-get update && apt-get install nfs-server -y
mkdir -p /mnt/nfs
echo "192.168.1.10:/raid/nfs  /mnt/nfs  nfs  defaults  0  0" >> /etc/fstab
mount -a
mount -v
touch /mnt/nfs/test
```

---

# Настроить chrony
## ISP
```bash
apt-get update && apt-get install -y chrony
echo -e "server 127.0.0.1 iburst prefer\nhwtimestamp *\nlocal stratum 5\nallow all" > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
```
## HQ-RTR
```cisco
en
conf
ntp server 172.16.1.1
ntp timezone utc+5
end
wr
show ntp status
```
## BR-RTR
```cisco
en
conf
ntp server 172.16.2.1
ntp timezone utc+5
end
wr
show ntp status
```
## HQ-SRV
```bash
apt-get install -y chrony
echo "server 172.16.1.1 iburst prefer" > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
```
## HQ-CLI
```bash
apt-get install -y chrony
echo "server 172.16.4.1 iburst prefer" > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
```
## BR-SRV
```bash
apt-get install -y chrony
echo "server 172.16.2.1 iburst prefer" > /etc/chrony.conf
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources
```

---

# Сконфигурировать ansible на BR-SRV
## HQ-CLI
```bash
echo -e "Port 2026" >> /etc/openssh/sshd_config
sed -i '/WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/s/^#//g' /etc/sudoers
useradd sshuser -u 2026
usermod -aG wheel sshuser
echo -e "P@ssw0rd\nP@ssw0rd" | passwd sshuser
systemctl restart sshd network
```

## BR-SRV
```bash
apt-get update && apt-get install ansible sshpass -y
echo -e "[s]\nHQ-SRV ansible_host=192.168.1.10\nHQ-CLI ansible_host=192.168.2.10\n[s:vars]\nansible_user=sshuser\nansible_port=2026\nansible_password=P@ssw0rd\n[r]\nHQ-RTR ansible_host=192.168.1.1\nBR-RTR ansible_host=192.168.3.1\n[r:vars]\nansible_user=net_admin\nansible_password=P@ssw0rd\nansible_connection=network_cli\nansible_network_os=ios" > /etc/ansible/hosts
rm -f /etc/ansible/ansible.cfg
echo -e "[defaults]\ninterpreter_python=auto_silent\nhost_key_checking=false" > /etc/ansible/ansible.cfg
ansible all -m ping
```

---

# Развернуть веб-приложение в docker BR-SRV
## BR-SRV
```bash
apt-get update && apt-get install docker-compose docker-engine -y
systemctl enable --now docker
sleep 2
mount -o loop /dev/sr0
docker load < /media/ALTLinux/docker/site_latest.tar
docker load < /media/ALTLinux/docker/mariadb_latest.tar
echo -e "services:\n  db:\n    image: mariadb\n    container_name: db\n    environment:\n      MYSQL_ROOT_PASSWORD: Passw0rd\n      MYSQL_DATABASE: testdb\n      MYSQL_USER: test\n      MYSQL_PASSWORD: Passw0rd\n    volumes:\n      - db_data:/var/lib/mysql\n    restart: always\n\n  testapp:\n    image: site\n    container_name: testapp\n    environment:\n      DB_TYPE: maria\n      DB_HOST: db\n      DB_NAME: testdb\n      DB_USER: test\n      DB_PASS: Passw0rd\n      DB_PORT: 3306\n    ports:\n      - \"8080:8000\"\n    restart: always\n\nvolumes:\n  db_data:" > docker-compose.yaml
sleep 1
docker compose up -d && sleep 5 && docker exec -it db mysql -u root -p'Passw0rd' -e "CREATE DATABASE IF NOT EXISTS testdb; CREATE USER 'test'@'%' IDENTIFIED BY 'Passw0rd'; GRANT ALL PRIVILEGES ON testdb.* TO 'test'@'%'; FLUSH PRIVILEGES;"
```

---

# Развернуть веб-приложение HQ-SRV
## HQ-SRV
```bash
apt-get update && apt-get install apache2 php8.2 apache2-mod_php8.2 mariadb-server php8.2-{opcache,curl,gd,intl,mysqli,xml,xmlrpc,ldap,zip,soap,mbstring,json,xmlreader,fileinfo,sodium} -y
systemctl enable --now httpd2 mysqld
mysql_secure_installation << EOF

y
y
P@ssw0rd
P@ssw0rd
y
y
y
y
EOF
mkdir /test
mount -o loop /dev/sr0 /test
mysql -e "CREATE DATABASE webdb;"
mysql -e "CREATE USER 'webc'@'localhost' IDENTIFIED BY 'P@ssw0rd';"
mysql -e "GRANT ALL PRIVILEGES ON webdb.* TO 'webc'@'localhost';"
mysql -e "FLUSH PRIVILEGES;"
iconv -f UTF-16LE -t UTF-8 /test/web/dump.sql > /tmp/dump_utf8.sql
mariadb -u root -pP@ssw0rd webdb < /tmp/dump_utf8.sql
chmod 777 /var/www/html
cp /test/web/index.php /var/www/html
cp /test/web/logo.png /var/www/html
rm -f /var/www/html/index.html
chown apache2:apache2 /var/www/html
systemctl restart httpd2
sed -i "s/\$servername = .*;/\$servername = 'localhost';/" /var/www/html/index.php
sed -i "s/\$dbname = .*;/\$dbname = 'webdb';/" /var/www/html/index.php
sed -i "s/\$password = .*;/\$password = 'P@ssw0rd';/" /var/www/html/index.php
sed -i "s/\$username = .*;/\$username = 'webc';/" /var/www/html/index.php
```

---

# Сконфигурировать статическую трансляцию портов
## BR-RTR
```cisco
en
conf t
ip nat source static tcp 192.168.5.5 80 172.16.2.2 8080
ip nat source static tcp 192.168.5.5 2026 172.16.2.2 2026
end
wr mem
```
## HQ-RTR
```cisco
en
conf t
ip nat source static tcp 192.168.1.10 2026 172.16.1.2 2026
end
wr mem
```
# Настроить веб-сервер nginx как обратный прокси-сервер
## ISP
```bash
apt-get update && apt-get install nginx
rm -f /etc/nginx/sites-available.d/default.conf
cat << EOF > /etc/nginx/sites-available.d/proxy.conf
server {
        listen 80;
        server_name docker.au-team.irpo;
        location / {
                proxy_pass http://172.16.2.2:8080;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
        }
}
server {
        listen 80;
        server_name web.au-team.irpo;
        location / {
                proxy_pass http://172.16.1.2:8080;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
        }
}
EOF
ln -s /etc/nginx/sites-available.d/proxy.conf /etc/nginx/sites-enabled.d/
systemctl enable --now nginx
```
# Настроить web-based аутентификацию
## ISP
```bash
apt-get update && apt-get install apache2-htpasswd -y
htpasswd -bc /etc/nginx/.htpasswd WEB P@ssw0rd
cat << EOF > /etc/nginx/sites-available.d/proxy.conf
server {
        listen 80;
        server_name docker.au-team.irpo;
        location / {
                proxy_pass http://172.16.2.2:8080;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $remote_addr;
        }
}
server {
        listen 80;
        server_name web.au-team.irpo;
        auth_basic "";
        auth_basic_user_file /etc/nginx/.htpasswd;
        location / {
                proxy_pass http://172.16.1.2:8080;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $remote_addr;
        }
}
EOF
systemctl restart nginx
```
# Установка Яндекс браузера
## HQ-CLI
```bash
echo trying to download an app...
sleep 25
echo analysing the download proccess
sleep 10
echo search a need app
sleep 5
echo one more try...
sleep 2
echo success!
apt-get install yandex-browser -y
echo the hardest app was download!
```




