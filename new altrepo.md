# [1]
## FOR ISP
```bash
cp /etc/apt/sources.list.d/alt.list /etc/apt/sources.list.d/alt.list.bak
sed -i 's|^rpm.*ftp\.altlinux|# &|g' /etc/apt/sources.list.d/alt.list
cat >> /etc/apt/sources.list.d/alt.list <<EOF
rpm [p11] http://192.168.0.222/mirror p11/branch/x86_64 classic
rpm [p11] http://192.168.0.222/mirror p11/branch/noarch classic
rpm [p11] http://192.168.0.222/mirror p11/branch/x86_64-i586 classic
EOF
```
## FOR CLI & SRV'S
```bash
cp /etc/apt/sources.list.d/alt.list /etc/apt/sources.list.d/alt.list.bak
sed -i 's|^rpm.*ftp\.altlinux|# &|g' /etc/apt/sources.list.d/alt.list
cat >> /etc/apt/sources.list.d/alt.list <<EOF
rpm [p10] http://192.168.0.222/mirror p10/branch/x86_64 classic
rpm [p10] http://192.168.0.222/mirror p10/branch/noarch classic
rpm [p10] http://192.168.0.222/mirror p10/branch/x86_64-i586 classic
EOF
```
