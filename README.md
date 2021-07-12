# Установка EVE-ng

Документация - https://www.eve-ng.net/index.php/documentation/community-cookbook/

FAQ - https://www.eve-ng.net/index.php/faq/

1. Скачать и установить VMware Workstation Player. Virtualbox не поддерживается разработчиками EVE-ng.
https://www.vmware.com/go/downloadplayer

2. Скачать OVF образ виртуальной машины EVE-ng Community Edition
https://www.eve-ng.net/index.php/download/

3. Распаковать zip архив, открыть VMware Workstation, импортировать VM.
VMware Workstation -> File -> Open -> eve-ng.ovf


4. При первом входе настроить из консоли VM сетевые параметры - IP, маска, шлюз.

Логин/пароль: root/eve

Чтобы изменить сетевые настройки повторно:
```
rm -f /opt/ovf/.configured
su -
```
5. Устанавливаем DHCP сервер
```
apt-get update
apt-get -y install isc-dhcp-server vim-nox

cat << 'EOF' > /etc/dhcp/dhcpd.conf
# EVE-NG Cloud9-Inet Interface
subnet 192.168.255.0 netmask 255.255.255.0 {
range 192.168.255.10 192.168.255.240;
interface pnet9;
default-lease-time 600;
max-lease-time 7200;
option domain-name "lab.local";
option domain-name-servers 8.8.8.8;
option broadcast-address 192.168.255.255;
option subnet-mask 255.255.255.0;
option routers 192.168.255.1;
}

# EVE-NG Cloud8-mgmt Interface
subnet 192.168.250.0 netmask 255.255.255.0 {
range 192.168.250.10 192.168.250.240;
interface pnet8;
default-lease-time 600;
max-lease-time 7200;
option domain-name "lab.local";
option broadcast-address 192.168.250.255;
option subnet-mask 255.255.255.0;
}
EOF

```

6. 
7. 
8. 

