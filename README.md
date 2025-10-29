# Demo2025
## Преднастройка

##### На всех КРОМЕ CLI выключаем GNOME
```
sudo systemctl set-default multi-user.target
sudo reboot
```
##### Выключаем и вырезаем на ВСЕХ ВМ NetworkManager
```
systemctl stop NetworkManager
systemctl disable NetworkManager
systemctl mask NetworkManager
```
##### На всех КРОМЕ ISP настраиваем DNS
```
nano /etc/resolv.conf
nameserver 77.88.8.8
nameserver 1.1.1.1
```
#### На РОУТЕРАХ настроить ip_forward
```
echo net.ipv4.ip_forward=1 > /etc/sysctl.conf
```

|Маска подсети|Полная маска|Сколько адресов вмещает|
|---|---|---|
|/26|255.255.255.192|62|
|/27|255.255.255.224|30|
|/28|255.255.255.240|14|
|/29|255.255.255.248|6|




## Задание 1.  Произведите `базовую` настройку устройств
### Задание
- Настройте имена устройств согласно топологии. Используйте полное доменное имя
- На всех устройствах необходимо сконфигурировать IPv4
- IP-адрес должен быть из приватного диапазона, в случае, если сеть локальная, согласно RFC1918
- Локальная сеть в сторону HQ-SRV(VLAN100) должна вмещать не более 64 адресов
- Локальная сеть в сторону HQ-CLI(VLAN200) должна вмещать не более 16 адресов
- Локальная сеть в сторону BR-SRV должна вмещать не более 32 адресов
- Локальная сеть для управления(VLAN999) должна вмещать не более 8 адресов
- Сведения об адресах занесите в отчёт, в качестве примера используйте Таблицу 3
### Решение
#### Настройка имени
##### ISP
```
hostnamectl set-hostname isp.au-team.irpo
bash
```
##### HQ-RTR
```
hostnamectl set-hostname hq-rtr.au-team.irpo
bash
```
##### BR-RTR
```
hostnamectl set-hostname br-rtr.au-team.irpo
bash
```
##### HQ-SRV
```
hostnamectl set-hostname hq-srv.au-team.irpo
bash
```
##### HQ-CLI
```
hostnamectl set-hostname hq-cli.au-team.irpo
bash
```
##### BR-SRV
```
hostnamectl set-hostname br-srv.au-team.irpo
bash
```

#### Настройка адрессации
##### ISP
```
auto ens224
iface ens224 inet static
address 172.16.4.1/28
auto ens256
iface ens256 inet static
address 172.16.5.1/28
```
##### HQ-RTR
```
allow-hotplug ens192
iface ens192 inet static
address 172.16.4.2/28
gateway 172.16.4.1

auto gre1
iface gre1 inet tunnel
address 172.16.0.1
netmask 255.255.255.252
mode gre
local 172.16.4.2
endpoint 172.16.5.2
ttl 64
```
##### BR-RTR
```
allow-hotplug ens192
iface ens192 inet static
address 172.16.5.2/28
gateway 172.16.5.1

auto ens224
iface ens224 inet static
address 192.168.0.1/27

auto gre1
iface gre1 inet tunnel
address 172.16.0.2
netmask 255.255.255.252
mode gre
local 172.16.5.2
endpoint 172.16.4.2
ttl 64
```
##### BR-SRV
```
allow-hotplug ens192
iface ens192 inet static
address 192.168.0.2/27
gateway 192.168.0.1
dns-nameservers 192.168.100.62 192.168.0.2
dns-search au-team.irpo
```
##### HQ-SRV
```
allow-hotplug ens192
iface ens192 inet static
address 192.168.100.62/26
gateway 192.168.100.1
```

#### Таблицы сетей
|   |   |   |
|---|---|---|
|**Сеть**|**Адрес подсети**|**Пул-адресов**|
|SRV-Net (VLAN 100)|192.168.100.0/26|192.168.100.1-62|
|CLI-Net (VLAN 200)|192.168.200.0/28|192.168.200.1-14|
|MGMT (VLAN 999)|192.168.99.0/29|192.168.99.1-6|
|BR-Net|192.168.0.0/27|192.168.0.1-30|
|ISP-HQ|172.16.4.0/28|172.16.4.1 - 14|
|ISP-BR|172.16.5.0/28|172.16.5.1 - 14|

Таблица подсетей

|   |   |   |   |   |   |
|---|---|---|---|---|---|
|**Устройство**|**Интерфейс**|**IPv4/IPv6**|**Маска/Префикс**|**Шлюз**|**Сеть**|
|ISP|ens192|(DHCP)|(DHCP)|(DHCP)|INTERNET|
|ens224|172.16.4.1|/28||ISP-HQ-RTR|
|ens256|172.16.5.1|/28||ISP-BR-RTR|
|HQ-RTR|ens192|172.16.4.2|/28|172.16.4.1|ISP-HQ-RTR|
|ens224.200|192.168.200.1|/28||HQ-RTR-CLI|
|ens224.100|192.168.100.1|/26||HQ-RTR-SRV|
|BR-RTR|ens192|172.16.5.2|/28|172.16.5.1|ISP-BR-RTR|
|ens224|192.168.0.1|/27||BR-RTR-SRV|
|HQ-SRV|ens192|192.168.100.62|/26|192.168.100.1|HQ-RTR-SRV|
|BR-SRV|ens192|192.168.0.2|/27|192.168.0.1|BR-RTR-SRV|
|HQ-CLI|ens192|192.168.200.##(DHCP)|/28|192.168.200.1|HQ-RTR-CLI|
              
Таблица адресации
