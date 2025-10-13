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
systemctl restart network
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

# HQ-RTR
## Базовая настройка
```cisco
en
conf t
hostname hq-rtr
ip domain-name au-team.irpo
int int0
ip addr 172.16.1.2/28
exit
port te0
service-instance te0/int0
encapsulation untagged
connect ip interface int0
exit
exit
int int1
ip addr 192.168.1.1/27
exit
int int2
ip addr 192.168.2.1/28
exit
port te1
service-instance te1/int1
encapsulation dot1q 100
rewrite pop 1
connect ip interface int1
exit
service-instance te1/int2
encapsulation dot1q 200
rewrite pop 1
connect ip interface int2
exit
exit
ip route 0.0.0.0 0.0.0.0 172.16.1.1
end
wr mem
```
## Создание локальных учётных записей
```cisco
en
conf t
username net_admin
role admin
password P@ssw0rd
end
wr mem
```
## Настройка коммутации в сегменте HQ
```cisco
en
conf t
int int3
ip addr 192.168.9.1/29
exit
port te1
service-instance te1/int3
encapsulation dot1q 999
rewrite pop 1
connect ip interface int3
end
wr mem
```
## IP-туннель между HQ и BR
```cisco
en
conf t
int tunnel.0
ip addr 172.16.0.1/30
ip mtu 1400
ip tunnel 172.16.1.2 172.16.2.2 mode gre
ip ospf authentication-key ecorouter
end
wr mem
```
## Обеспечение динамической маршрутизации
```cisco
en
conf t
router ospf 1
network 172.16.0.0/30 area 0
network 192.168.1.0/27 area 0
network 192.168.2.0/28 area 0
passive-interface default
no passive-interface tunnel.0
area 0 authentication
exit
wr mem
```
## Настройка динамической трансляции адресов
```cisco
en
conf t
interface int1
ip nat inside
exit
interface int2
ip nat inside
exit
interface int0
ip nat outside
exit
ip name-server 8.8.8.8
ip nat pool np 192.168.1.1-192.168.1.254,192.168.2.1-192.168.2.254
ip nat source dynamic inside-to-outside pool np overload interface int0
end
wr mem
```
## Настройка протокола динамической конфигурации хостов
```cisco
en
conf t
ip pool dp 192.168.2.10-192.168.2.10 
dhcp-server 1 
pool dp 1 
mask 255.255.255.240 
gateway 192.168.2.1 
dns 192.168.1.10 
domain-name au-team.irpo 
exit
exit
interface int2 
dhcp-server 1 
exit
wr mem
```
## Настроить часовой пояс на всех устройствах
```cisco
en 
conf t 
ntp timezone utc+5 
exit
```

---

# BR-RTR
## Базовая настройка
```cisco
en
conf t
hostname hq-rtr
ip domain-name au-team.irpo
int int0
ip addr 172.16.2.2/28
exit
port te0
service-instance te0/int0
encapsulation untagged
connect ip interface int0
exit
exit
int int1
ip addr 192.168.3.1/28
exit
port te1
service-instance te1/int1
encapsulation untagged
connect ip interface int1
exit
exit
ip route 0.0.0.0 0.0.0.0 172.16.2.1
end
wr mem
```
## Создание локальных учётных записей
```cisco
en
conf t
username net_admin
role admin
password P@ssw0rd
end
wr mem
```
## IP-туннель между HQ и BR
```cisco
en
conf t
int tunnel.0
ip addr 172.16.0.2/30
ip mtu 1400
ip tunnel 172.16.2.2 172.16.1.2 mode gre
ip ospf authentication-key ecorouter
end
wr mem
```
## Обеспечение динамической маршрутизации
```cisco
en
conf t
router ospf 1
network 172.16.0.0/30 area 0
network 192.168.3.0/28 area 0
passive-interface default
no passive-interface tunnel.0
area 0 authentication
exit
wr mem
```
## Настройка динамической трансляции адресов
```cisco
en
conf t
interface int1
ip nat inside
exit
interface int0
ip nat outside
exit
ip name-server 8.8.8.8
ip nat pool np 192.168.3.1-192.168.3.254
ip nat source dynamic inside-to-outside pool np overload interface int0
end
wr mem
```
## Настроить часовой пояс на всех устройствах
```cisco
en 
conf t 
ntp timezone utc+5 
exit
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
systemctl restart network 
```
## Создание локальный учетных записей (НЕ ПОЛНОСТЬЮ АВТОМАТИЗИРОВАННО!)
```bash
sed -i '/WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/s/^#//g' /etc/sudoers
useradd sshuser -u 2026
usermod -aG root sshuser
echo -e "P@ssw0rd\nP@ssw0rd" | passwd sshuser
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
echo -e "address=/br-srv.au-team.irpo/192.168.3.10" >> /etc/dnsmasq.conf
echo -e "address=/docker.au-team.irpo/172.16.2.1\naddress=/web.au-team.irpo/172.16.1.1" >> /etc/dnsmasq.conf
```
## Настроить часовой пояс на всех устройствах
```bash
timedatectl set-timezone Asia/Yekaterinburg 
```

---

# BR-SRV
## Базовая настройка
```bash
hostnamectl set-hostname br-srv.au-team.irpo; exec bash
mkdir /etc/net/ifaces/ens20
echo -e "DISABLED=no\nTYPE=eth\nBOOTPROTO=static\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens20/options
echo 192.168.3.10/28 > /etc/net/ifaces/ens20/ipv4address
echo default via 192.168.3.1 > /etc/net/ifaces/ens20/ipv4route
systemctl restart network 
```
## Создание локальный учетных записей (НЕ ПОЛНОСТЬЮ АВТОМАТИЗИРОВАННО!)
```bash
sed -i '/WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL/s/^#//g' /etc/sudoers
useradd sshuser -u 2026
usermod -aG root sshuser
echo -e "P@ssw0rd\nP@ssw0rd" | passwd sshuser
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
## Настроить часовой пояс на всех устройствах
```bash
timedatectl set-timezone Asia/Yekaterinburg 
```

---

# HQ-CLI
## Базовая настройка
```bash
hostnamectl set-hostname hq-cli.au-team.irpo; exec bash
rm -rf /etc/net/ifaces/ens18
mkdir /etc/net/ifaces/ens20
echo -e "DISABLED=no\nTYPE=eth\nBOOTPROTO=static\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens20/options
echo 192.168.2.10/28 > /etc/net/ifaces/ens20/ipv4address
echo default via 192.168.2.1 > /etc/net/ifaces/ens20/ipv4route
```
## Настройка динамической трансляции адресов
```bash
echo nameserver 8.8.8.8 > /etc/resolv.conf
```
## Настройка протокола динамической конфигурации хостов
```bash
echo -e "DISABLED=no\nTYPE=eth\nBOOTPROTO=dhcp\nCONFIG_IPV4=yes" > /etc/net/ifaces/ens20/options
systemctl restart network 
```
## Настроить часовой пояс на всех устройствах
```bash
timedatectl set-timezone Asia/Yekaterinburg 
```
