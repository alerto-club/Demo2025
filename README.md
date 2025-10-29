![[Pasted image 20251029033949.png]]

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

#### Настройка адресации
##### ISP
```
auto ens18
iface ens18 inet dhcp

auto ens19
iface ens19 inet static
address 172.16.1.1/28

auto ens20
iface ens20 inet static
address 172.16.2.1/28
```
##### HQ-RTR
```
allow-hotplug ens18
iface ens18 inet static
address 172.16.1.2/28
gateway 172.16.1.1

#vlan 100
auto ens19.100
iface ens19.100 inet static
address 192.168.0.1
netmask 255.255.255.224
vlan_raw_device ens19

#vlan 200
auto ens19.200
iface ens19.200 inet static
address 192.168.0.33
netmask 255.255.255.240
vlan_raw_device ens19

#vlan 999
auto ens19.999
iface ens19.999 inet static
address 192.168.0.49
netmask 255.255.255.248
vlan_raw_device ens19

#gre
auto gre1
iface gre1 inet tunnel
address 172.16.0.1
netmask 255.255.255.252
mode gre
local 172.16.1.2
endpoint 172.16.2.2
ttl 64
```
##### BR-RTR
```
allow-hotplug ens18
iface ens18 inet static
address 172.16.2.2/28
gateway 172.16.2.1

auto ens19
iface ens19 inet static
address 192.168.1.1/27

auto gre1
iface gre1 inet tunnel
address 172.16.0.2
netmask 255.255.255.252
mode gre
local 172.16.2.2
endpoint 172.16.1.2
ttl 64
```
##### BR-SRV
```
allow-hotplug ens18
iface ens18 inet static
address 192.168.1.2/27
gateway 192.168.1.1
dns-nameservers 192.168.100.62 192.168.1.2
dns-search au-team.irpo
```
##### HQ-SRV
```
allow-hotplug ens18
iface ens18 inet static
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

## Задание 2 `[NAT на ISP]`
### Задание
- Настройте адресацию на интерфейсах:
    
    - Интерфейс, подключенный к магистральному провайдеру, получает адрес по DHCP [Выполнено в задании 1]
        
    - Настройте маршруты по умолчанию там, где это необходимо
        
    - Интерфейс, к которому подключен HQ-RTR, подключен к сети 172.16.4.0/28 [Выполнено в задании 1]
        
    - Интерфейс, к которому подключен BR-RTR, подключен к сети 172.16.5.0/28 [Выполнено в задании 1]
        
    - На ISP настройте динамическую сетевую трансляцию в сторону HQ-RTR и BR-RTR для доступа к сети Интернет
### Решение
#### Настройка динамической сетевой трансляции на _`ISP`_
```
apt install iptables iptables-persistent –y
```

```
iptables –t nat –A POSTROUTING –s 172.16.1.0/28 –o ens18 –j MASQUERADE  
iptables –t nat –A POSTROUTING –s 172.16.2.0/28 –o ens18 –j MASQUERADE
netfilter-persistent save
systemctl restart netfilter-persistent  
```

## Задание 3 `[Создание учетных записей]`

### Создание локальных учетных записей

### Задание
- Создайте пользователя remote_user на серверах HQ-SRV и BR-SRV
    
    - Пароль пользователя remote_user с паролем P@ssw0rd
        
    - Идентификатор пользователя 2026
        
    - Пользователь remote_user должен иметь возможность запускать sudo без дополнительной аутентификации.
        
- Создайте пользователя net_admin на маршрутизаторах HQ-RTR и BR-RTR
    
    - Пароль пользователя net_admin с паролем P@ssw0rd
        
    - При настройке ОС на базе Linux, запускать sudo без ввода пароля 
		   
    - При настройке ОС отличных от Linux пользователь должен обладать максимальными привилегиями.

### Решение
#### remote_user
**1.** Создаём remote_user следующими командами:
```
useradd remote_user -u 2026
passwd remote_user
P@ssw0rd
```
 
  **2.** После чего даем пользователю **`root`** права:
```yaml
usermod -aG sudo remote_user
```

**3.** Добавляем следующую строку в файле `/etc/sudoers`:
```yaml
remote_user ALL=(ALL) NOPASSWD:ALL
```

**4.** Создаем и задаем необходимые права на домашнюю папку
```
mkdir /home/remote_user
chown remote_user:remote_user /home/remote_user
chmod 700 /home/remote_user
```

#### net_admin

**1.** Создаём **`net_admin`**, следующими командами, но уже без `-u 1010` и с новым паролем:
```
useradd net_admin
passwd net_admin
P@ssword
```

**2.** После чего даем пользователю **`root`** права:
```yaml
usermod -aG sudo net_admin
```

  **3.** Добавляем следующую строку в `/etc/sudoers`:
```yaml
net_admin ALL=(ALL) NOPASSWD:ALL
```

**4.** Создаем и задаем необходимые права на домашнюю папку
```
mkdir /home/net_admin
chown net_admin:net_admin /home/net_admin
chmod 700 /home/net_admin
```

## Задание 4 `[VLAN]`
### Задание
#### Настройте на интерфейсе `HQ-RTR` в сторону офиса `HQ` виртуальный коммутатор
- Сервер HQ-SRV должен находиться в ID VLAN 100
- Клиент HQ-CLI в ID VLAN 200
- Создайте подсеть управления с ID VLAN 999
- Основные сведения о настройке коммутатора и выбора реализации разделения на VLAN занесите в отчёт
### Решение
```
apt install vlan
```

```
echo 8021q >> /etc/modules
```

## Задание 5 `[SSH]`
### Задание
#### Настройка безопасного удаленного доступа на серверах `HQ-SRV` и `BR-SRV`
- Для подключения используйте порт 2026
- Разрешите подключения только пользователю sshuser
- Ограничьте количество попыток входа до двух
- Настройте баннер «Authorized access only»
### Решение
```
apt-get install openssh-server -y
```

```
nano /etc/ssh/sshd_config
```

```
Port 2026
MaxAuthTries 2
PasswordAuthentication yes
Banner /etc/ssh/banner
AllowUsers  sshuser
           ^ - это TAB
```

```
nano /etc/ssh/banner
```

```
----------------------
Authorized access only
----------------------
```

```
systemctl restart sshd
```

## Задание 6 `[GRE-TUNNEL]`
### Задание
#### Между офисами `HQ` и `BR` необходимо сконфигурировать _`IP-туннель`_
- Сведения о туннеле занесите в отчёт
- На выбор технологии GRE или IP in IP

### Решение
Для работы туннеля необходимо добавить строчку в файл на HQ-RTR и BR-RTR `/etc/modules`

```
echo gre_ip >> /etc/modules
```

```
systemctl restart networking
```

## Задание 7 `[OSPF]`
### Задание
#### Обеспечьте динамическую маршрутизацию: ресурсы одного офиса должны быть доступны из другого офиса. Для обеспечения динамической маршрутизации используйте `link state` протокол на ваше усмотрение
- Разрешите выбранный протокол только на интерфейсах в ip туннеле
- Маршрутизаторы должны делиться маршрутами только друг с другом
- Обеспечьте защиту выбранного протокола посредством парольной защиты
- Сведения о настройке и защите протокола занесите в отчёт
### Решение
#### HQ-RTR
**1.** Устанавливаем пакет `FRR`
```
sudo apt install -y frr
```

**2.** В конфигурационном файле `/etc/frr/daemons` необходимо активировать выбранный протокол `OSPF` для дальнейшей реализации его настройки:
```
nano /etc/frr/daemons

!!! Ищем следующую строку и меняем с (no) на (yes)
ospfd = yes

```

**3.** Далее перезаргужаем и добавляем в автозагрузку службу **`FRR`**
```
systemctl restart frr
systemctl enable --now frr
```

**4.** Переходим в интерфейс управления симуляцией **`FRR`** командой:
```
vtysh
```

**5.** Пишем команды для настройки **маршрутизации:**
```
conf t
router ospf
  passive-interface default
  router-id 1.1.1.1
  network 172.16.0.0/30 area 0
  network 192.168.0.0/27 area 1
  network 192.168.0.33/28 area 2
  network 192.168.0.48/29 area 3
  area 0 authentication
exit

int gre1
  no ip ospf network broadcast
  no ip ospf passive
  ip ospf authentication
  ip ospf authentication-key password
(config-if)exit
(config)exit
#write
```

#### BR-RTR
**1-4.** Пункты такие же как и в HQ-RTR

**5.** Пишем команды для настройки **маршрутизации:**
**Меняется:**
- `id-router: 2.2.2.2`
- `network 192.168.1.0/27 area 4`
- `network 172.16.0.0/30 area 0`
    
```
conf t
router ospf
  passive-interface default
  router-id 2.2.2.2
  network 192.168.0.1/27 area 4
  network 172.16.0.0/30 area 0
  area 0 authentication
exit

int gre1
  no ip ospf network broadcast
  no ip ospf passive
  ip ospf authentication
  ip ospf authentication-key password
(config-if)exit
(config)exit
#write
```

#### ПРОВЕРКА

Пингуем: **`BR-SRV - > HQ-SRV`** и **`BR-SRV - > HQ-CLI`**

Проверка в **FRR**:
```
vtysh
  show ip ospf neighbor
  show ip route ospf
```

## Задание 8 `[NAT на HQ-rtr и BR-rtr]`
### Задание
### Настройка динамической трансляции адресов
- Настройте динамическую трансляцию адресов для обоих офисов.
- Все устройства в офисах должны иметь доступ к сети Интернет
### Решение
#### Настройка динамической сетевой трансляции на `HQ-RTR`
```
apt-get install iptables iptables-persistent –y
iptables –t nat –A POSTROUTING –s 192.168.0.0/27 –o ens18 –j MASQUERADE
iptables –t nat –A POSTROUTING –s 192.168.0.33/28 –o ens18 –j MASQUERADE
iptables –t nat –A POSTROUTING –s 192.168.0.48/29 –o ens18 –j MASQUERADE
netfilter-persistent save
systemctl restart netfilter-persistent  
```

#### Настройка динамической сетевой трансляции на `BR-RTR`
```
apt-get install iptables iptables-persistent –y
iptables –t nat –A POSTROUTING –s 192.168.1.0/27 –o ens18 –j MASQUERADE
netfilter-persistent save
systemctl restart netfilter-persistent  
```

## Задание 9 `[DHCP]`
### Задание
#### Настройка протокола динамической конфигурации хостов
- Настройте нужную подсеть
- Для офиса HQ в качестве сервера DHCP выступает маршрутизатор HQ-RTR.
- Клиентом является машина HQ-CLI.
- Исключите из выдачи адрес маршрутизатора
- Адрес шлюза по умолчанию – адрес маршрутизатора HQ-RTR.
- Адрес DNS-сервера для машины HQ-CLI – адрес сервера HQ-SRV.
- DNS-суффикс для офисов HQ – au-team.irpo
- Сведения о настройке протокола занесите в отчёт
### Решение
**1.** Устанавливаем сам **DHCP-сервер**:
```
apt install isc-dhcp-server
```

  **2.** После чего переходим в конфигурацию файла `dhcpd.conf` и добавляем следующие строчки:
```
nano /etc/dhcp/dhcpd.conf
```

```
subnet 192.168.200.0 netmask 255.255.255.240 {
  range 192.168.200.2 192.168.200.14;
  option domain-name-servers 192.168.100.62;
  option domain-name "au-team.irpo";
  option routers 192.168.200.1;
  default-lease-time 600;
  max-lease-time 7200;
}
```

  **3.** После чего переходим в конфигурацию файла `isc-dhcp-server` и меняем ее добалвяя данный текст:
```
nano /etc/default/isc-dhcp-server

INTERFACESv4="ens224:1" - порт смотрящий в сторону CLI
```

  **4.** Включаем сервиc **`DHCP`** и добавляем в автозагрузку на **`HQ-RTR`**:
```
systemctl start isc-dhcp-server
systemctl enable isc-dhcp-server
```

**5.** Далее на клиенсткой машине необходимо в настройках сет.адаптера выбрать режим **DHCP** и проверить работоспособность

## Задание 10 `[DNS]`
### Задание
#### Настройка DNS для офисов HQ и BR
- Основной DNS-сервер реализован на HQ-SRV.
- Сервер должен обеспечивать разрешение имён в сетевые адреса устройств и обратно в соответствии с таблицей 2
- В качестве DNS сервера пересылки используйте любой общедоступный DNS сервер

|   |   |   |
|---|---|---|
|Устройство|Запись|Тип|
|HQ-RTR|hq-rtr.au-team.irpo|A,PTR|
|BR-RTR|br-rtr.au-team.irpo|A|
|HQ-SRV|hq-srv.au-team.irpo|A,PTR|
|HQ-CLI|hq-cli.au-team.irpo|A,PTR|
|BR-SRV|br-srv.au-team.irpo|A|
|HQ-RTR|moodle.au-team.irpo|CNAME|
|BR-RTR|wiki.au-team.irpo|CNAME|

Таблица 2
### Решение
#### HQ-SRV 
**1.** Для работы с **DNS** требуется установить **`bind`** и доп. пакет командой:
```
apt-get install bind9 bind9-utils
```

  **2.** Далее необходимо сконфигурировать файл **`named.conf.options`** таким образом:
```
nano /etc/bind/named.conf.options
```

```
  listen-on { 127.0.0.1; 192.168.100.0/26; 192.168.200.0/28; 192.168.0.0/27; 172.16.0.0/30; };
  forwarders { 127.0.0.1; 8.8.8.8; 192.168.100.62; 8.8.4.4; };
  recursion yes;
  allow-query { 127.0.0.1; 192.168.100.0/26; 192.168.200.0/28; 192.168.0.0/27; 172.16.0.0/30; };
  allow-query-cache { 127.0.0.1; 192.168.100.0/26; 192.168.200.0/28; 192.168.0.0/27; 172.16.0.0/30; };
  allow-recursion { 127.0.0.1; 192.168.100.0/26; 192.168.200.0/28; 192.168.0.0/27; 172.16.0.0/30; };
  dnssec-validation auto;
```

**3.** Конфигурация ключей **`rndc`**:
```
rndc-confgen > /etc/rndc.key 
```

  **❗ Для проверки, можно использовать комманду:**
```
named-checkconf
```

  **4.** Далее необходимо запустить **утилиту** коммандой:
```
systemctl enable --now named
```

  **5.** Далее требуется изменить конфигурацию файла на **`HQ-SRV`** **`resolv.conf`**:
```
nano /etc/resolv.conf
```

```
nameserver 192.168.100.62
search au-team.irpo
```

**6.** После чего требуется прописать в **`/etc/bind/named.conf.local`**:
```
zone "au-team.irpo" {
  type master;
  file "/etc/bind/au-team.irpo";
};

zone "100.168.192.in-addr.arpa" {
  type master;
  file "/etc/bind/100.168.192.in-addr.arpa";
};

zone "200.168.192.in-addr.arpa" {
  type master;
  file "/etc/bind/200.168.192.in-addr.arpa";
};

zone "0.168.192.in-addr.arpa" {
  type master;
  file "/etc/bind/0.168.192.in-addr.arpa";
};
```

  **7.** Далее следующими командами **создаём копию** файла и присваиваем права:
```
cp /etc/bind/db.local /etc/bind/au-team.irpo
```

  **8.** После чего приводим **файл `au-team.irpo`** к следующему виду:
```
nano /etc/bind/au-team.irpo
```

```
$TTL    1D
@       IN      SOA     hq-srv.au-team.irpo. root.au-team.irpo. (
                                2024102200      ; serial
                                12H             ; refresh
                                1H              ; retry
                                1W              ; expire
                                1H              ; ncache
                        )
@       IN      NS      hq-srv.au-team.irpo.
hq-rtr  IN      A       192.168.100.1
br-rtr  IN      A       192.168.0.1
hq-srv  IN      A       192.168.100.62
hq-cli  IN      A       192.168.200.3
br-srv  IN      A       192.168.0.2
moodle  IN      CNAME   hq-rtr
wiki    IN      CNAME   br-rtr
_ldap._tcp.au-team.irpo. IN SRV 0 5 389 br-srv.au-team.irpo.
_kerberos._tcp.au-team.irpo.        IN      SRV     0 100 88  br-srv.au-team.irpo.
_kdc._tcp.au-team.irpo.             IN      SRV     0 100 88  br-srv.au-team.irpo.
_kpasswd._tcp.au-team.irpo.         IN      SRV     0 100 464 br-srv.au-team.irpo.
```

  
**9.** После чего **создаем файлы** командами:
```
cp /etc/bind/db.127 /etc/bind/100.168.192.in-addr.arpa
cp /etc/bind/db.127 /etc/bind/200.168.192.in-addr.arpa
cp /etc/bind/db.127 /etc/bind/0.168.192.in-addr.arpa
```

**10.** После изменений файл **`100.168.192.in-addr.arpa`** выглядит так:
```
nano /etc/bind/100.168.192.in-addr.arpa
```

```
$TTL    1D
@       IN      SOA     hq-srv.au-team.irpo. root.au-team.irpo. (
                                2024102200      ; Serial
                                12H             ; Refresh
                                1H              ; Retry
                                1W              ; Expire
                                1H              ; Ncache
                        )
@       IN      NS      localhost.
1       IN      PTR     hq-rtr.au-team.irpo.
62      IN      PTR     hq-srv.au-team.irpo.
```

**11.** После изменений файл **`200.168.192.in-addr.arpa`** выглядит так:
```
nano /etc/bind/200.168.192.in-addr.arpa
```

```
$TTL    1D
@       IN      SOA     hq-srv.au-team.irpo. root.au-team.irpo. (
                                2024102200      ; Serial
                                12H             ; Refresh
                                1H              ; Retry
                                1W              ; Expire
                                1H              ; Ncache
                        )
@       IN      NS      localhost.
2       IN      PTR     hq-cli.au-team.irpo.
```

  **12.** После изменений файл **`0.168.192.in-addr.arpa`** выглядит так:
```
nano /etc/bind/0.168.192.in-addr.arpa
```

```
$TTL    1D
@       IN      SOA     hq-srv.au-team.irpo. root.au-team.irpo. (
                                2024102200      ; Serial
                                12H             ; Refresh
                                1H              ; Retry
                                1W              ; Expire
                                1H              ; Ncache
                        )
@       IN      NS      localhost.
1       IN      PTR     br-rtr.au-team.irpo.
2       IN      PTR     br-srv.au-team.irpo.
```

 ❗ ❗ ❗ Все пробелы выше ^ ставятся TAB'ом
  
**13.** После чего можно проверить **ошибки** командой:
```
named-checkconf -z
```

  **14.** А также перезапускаем **`bind`** командой:
```
systemctl restart named bind9
```

  **15.** На всех устройствах локальной сети необходимо указать в конфигурационном файле `resolv.conf`:
```
nano /etc/resolv.conf
```

```
nameserver 192.168.100.62
search au-team.irpo
```

  **16.** Проверить работоспособность можно **командой**:
```
nslookup **IP-адрес/DNS-имя**
```

## Задание 11 `[ВРЕМЯ/ДАТА]`
### Задание
Настройте часовой пояс на всех устройствах, согласно месту проведения экзамена
### Решение
На Linux настраивается часовой пояс командой:
```
timedatectl set-timezone Asia/Moscow
```
