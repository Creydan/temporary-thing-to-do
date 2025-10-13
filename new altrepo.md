# [1]
## FOR ISP
```bash
cp /etc/apt/sources.list.d/alt.list /etc/apt/sources.list.d/alt.list.bak
sed -i 's|^rpm.*ftp\.altlinux|# &|g' /etc/apt/sources.list.d/alt.list
cat >> /etc/apt/sources.list.d/alt.list <<EOF
rpm [p11] http://192.168.0.91/mirror p11/branch/x86_64 classic
rpm [p11] http://192.168.0.91/mirror p11/branch/noarch classic
rpm [p11] http://192.168.0.91/mirror p11/branch/x86_64-i586 classic
EOF
```
## FOR CLI & SRV'S
```bash
cp /etc/apt/sources.list.d/alt.list /etc/apt/sources.list.d/alt.list.bak
sed -i 's|^rpm.*ftp\.altlinux|# &|g' /etc/apt/sources.list.d/alt.list
cat >> /etc/apt/sources.list.d/alt.list <<EOF
rpm [p10] http://192.168.0.91/mirror p10/branch/x86_64 classic
rpm [p10] http://192.168.0.91/mirror p10/branch/noarch classic
rpm [p10] http://192.168.0.91/mirror p10/branch/x86_64-i586 classic
EOF
```
---
# [2]
## ISP
```bash
apt-get update && apt-get install chrony nginx iptables apache2-htpasswd -y
apt-get reinstall tzdata -y
iptables -t nat -A POSTROUTING -o ens20 -s 0/0 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
```
## HQ-SRV
```bash
apt-get update && apt-get install chrony nginx iptables apache2-htpasswd -y
apt-get reinstall tzdata -y
```
## HQ-CLI
```bash
apt-get update && apt-get install chrony nginx iptables apache2-htpasswd -y
apt-get reinstall tzdata -y
```
## BR-SRV
```bash
apt-get update && apt-get install chrony ansible task-samba-dc docker-engine docker-compose -y
```
