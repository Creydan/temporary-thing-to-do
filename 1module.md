# HQ-SRV
## Базовая комутация
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
