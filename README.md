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
# не делайте apt upgrade, это может поломать пакеты EVE-ng
apt-get -y install isc-dhcp-server vim-nox
```
6. Настраиваем сети для cloud-9(DHCP+Inet), Cloud-8(DHCP) и ребутаем сервер

Cloud-9 и Cloud-8 - это устройства типа "облако", которые в дальнейшем будем использовать в веб-интерфейсе EVE-ng

Cloud-9 - с доступом в интернет

Cloud-8 - без доступа в интернет
```
vim /etc/network/interfaces
# в конце файла ищем строчки и меняем конфиг на:
iface eth8 inet manual
auto pnet8
iface pnet8 inet static
    address 192.168.250.1
    netmask 255.255.255.0
    bridge_ports eth8
    bridge_stp off

iface eth9 inet manual
auto pnet9
iface pnet9 inet static
    address 192.168.255.1
    netmask 255.255.255.0
    bridge_ports eth9
    bridge_stp off

# сохраняем файл и ребутаем сервер
reboot
```
7. Настраваем конфиг для DHCP сервера
```
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

# добавляем в автозагрузку и стартуем сервис
systemctl enable isc-dhcp-server
systemctl restart isc-dhcp-server
systemctl status isc-dhcp-server

# для удобства добавляем скрипт, который выводит список IP выданных DHCP сервером
mkdir /root/scripts && cd /root/scripts
wget https://raw.githubusercontent.com/svmyasnikov/eve-ng/main/dhcp-lease.py
chmod +x /root/scripts/dhcp-lease.py
ln -sf /root/scripts/dhcp-lease.py /usr/local/bin/dhcp-lease

# проверяем работу скрипта
dhcp-lease
```

6. Включаем IP Routing и NAT для Cloud-9.
```
# проверяем что ipv4 роутинг в ядре выключен, значение = 0
cat /proc/sys/net/ipv4/ip_forward

# включаем роутинг
sysctl -w net.ipv4.ip_forward=1

# включаем роутинг постоянно
vim /etc/sysctl.conf
# ищем параметр ipv4.ip_forward и меняем "0" на "1"

# Включаем NAT для Cloud-9 сети
iptables -t nat -A POSTROUTING -o pnet0 -s 192.168.255.0/24 -j MASQUERADE

# Сохраняем правила iptables
apt-get install iptables-persistent
netfilter-persistent save
netfilter-persistent reload
```
7. Скачиваем ISO образ ubuntu-20 server и готовим файлы для qemu VM.
https://ubuntu.com/download/server

```
wget https://releases.ubuntu.com/20.04.2/ubuntu-20.04.2-live-server-amd64.iso

# создаем папку для образа qemu VM
mkdir /opt/unetlab/addons/qemu/linux-ubuntu-20

# ISO образ перемещаем в папку VM и меняем имя на cdrom.iso
mv ./ubuntu*.iso /opt/unetlab/addons/qemu/linux-ubuntu-20/cdrom.iso

# создаем виртуальный диск на 10Gb
cd /opt/unetlab/addons/qemu/linux-ubuntu-20
/opt/qemu/bin/qemu-img create -f qcow2 virtioa.qcow2 10G

# после работы с файлами на EVE-ng сервере нужно запустить скрипт для установки корректных прав
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```
8. Переходим в web-интерфейс EVE-ng. Логин/пароль - admin/eve

IP EVE-ng можно проверить командой:
```
ifconfig eth0
```
9. Добавляем устройство - тип Linux, RAM уменьшаем до 1Gb. После установки можно уменьшить RAM до 512Mb.
10. Добавляем облако Cloud-9, подключаем к linux устройству.
11. Запускаем Linux устройство, приступаем к установке. Устанавливаем минимальный набор пакетов. Выбираем только Openssh-server.
12. После завершения установки выключаем Linux устройство.
13. Возвращаемся в консоль EVE-ng. Удаляем ISO и сохраняем изменения на диск qemu.
```
rm /opt/unetlab/addons/qemu/linux-ubuntu-20/cdrom.iso

# Смотрим имя папки с временными файлами:
ls /opt/unetlab/tmp/0/

# имя папки генерируется случайно! скопируйте имя которое будет у вас. 
# Формат /opt/unetlab/tmp/POD_ID/RANDOM_DIGITS/NODE_ID/
cd /opt/unetlab/tmp/0/a0f4a75b-964a-4db5-82d1-d54c124f1871/1/

# внутри папки будет временный файл диска qemu. Проверяем какой VM принадлежит файл и сохраняем файл в постоянный образ.
/opt/qemu/bin/qemu-img info virtioa.qcow2
/opt/qemu/bin/qemu-img commit virtioa.qcow2

# не забываем исправить права к файлам
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```
14. Образ qemu VM с Ubuntu готов. Можно добавить в лабу несколько новых устройств типа linux. При добавлении устройств уменьшите размер RAM, увеличьте кол-во сетевых интерфейсов.
