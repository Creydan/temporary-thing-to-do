# ISP
## Базовая настройка
```bash
hostnamectl set-hostname isp; exec bash
```
## Настройка доступа к Интернету
```bash
mkdir /etc/net/ifaces/ens2{0,1,2}
echo -e "DISABLED=no\nTYPE=eth\nBOOTPROTO=dhcp\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens20/options
echo -e "DISABLED=no\nTYPE=eth\nBOOTPROTO=static\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens21/options
echo -e "DISABLED=no\nTYPE=eth\nBOOTPROTO=static\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens22/options
echo 172.16.1.1/28 > ens21/ipv4address
echo 172.16.2.1/28 > ens22/ipv4address
sed -i '/net.ipv4.ip_forward/s/0/1/g' /etc/net/sysctl.conf
apt-get update && apt-get install iptables -y
iptables -t nat -A POSTROUTING -o ens20 -s 0/0 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
systemctl restart network iptables
```
## Настроить часовой пояс на всех устройствах
```bash
apt-get install tzdata
timedatectl set-timezone Asia/Yekaterinburg 
```
---
# HQ-SRV
## Базовая настройка
```bash
hostnamectl set-hostname hq-srv.au-team.irpo; exec bash
mkdir /etc/net/ifaces/ens20
echo -e "DISABLED=no\nTYPE=eth\nBOOTPROTO=static\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens20/options
echo 192.168.1.10/27 > /etc/net/ifaces/ens20/ipv4address
echo default via 192.168.1.1 > /etc/net/ifaces/ens20/ipv4route
```
## Создание локальный учетных записей
```bash
sed -i '/WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/s/^#//g' /etc/sudoers
useradd sshuser -u 2026
usermod -aG root sshuser
passwd sshuser
# ПАРОЛЬ ВВОДИТСЯ, АВТОМАТИЧЕСКИ ЭТОГО СДЕЛАТЬ ПОКА НЕ НАШЁЛ КАК
```
## Настройка безопасного удалённого доступа
```bash
echo -e "Port 2026\nAllowUsers sshuser\nMaxAuthTries 2\nBanner /etc/openssh/banner" >> /etc/openssh/sshd_config
echo "Authorized access only!" > /etc/openssh/banner
systemctl restart sshd
```
## Настройка динамической трансляции адресов
```bash
echo nameserver 8.8.8.8 > /etc/resolv.conf
```
## Настройте DNS-сервер на сервере HQ-SRV
```bash
systemctl disable --now bind
apt-get update
apt-get install dnsmasq -y
systemctl enable --now dnsmasq
sed -i '/^[@#]/ d' /etc/dnsmasq.conf && sed -i '/^$/d' /etc/dnsmasq.conf
echo -e "\nno-resolv\ndomain=au-team.irpo\nserver=8.8.8.8\ninterface=*\n" >> /etc/dnsmasq.conf
echo -e "address=/hq-rtr.au-team.irpo/192.168.1.1\nptr-record=1.1.168.192.in-addr.arpa,hq-rtr.au-team.irpo" >> /etc/dnsmasq.conf
echo -e "address=/br-rtr.au-team.irpo/192.168.2.1" >> /etc/dnsmasq.conf
echo -e "address=/hq-srv.au-team.irpo/192.168.1.10\nptr-record=10.1.168.192.in-addr.arpa,hq-srv.au-team.irpo" >> /etc/dnsmasq.conf
echo -e "address=/hq-cli.au-team.irpo/192.168.2.10\nptr-record=10.2.168.192.in-addr.arpa,hq-cli.au-team.irpo" >> /etc/dnsmasq.conf
echo -e "address=/br-srv.au-team.irpo/192.168.5.5" >> /etc/dnsmasq.conf
echo -e "address=/docker.au-team.irpo/172.16.2.1\naddress=/web.au-team.irpo/172.16.1.1" >> /etc/dnsmasq.conf
```
