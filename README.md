<img width="564" height="672" alt="Pasted image 20251029033949" src="https://github.com/alerto-club/Demo2025/blob/main/Pasted%20image%2020251029033949.png" />

## Преднастройка

##### На всех КРОМЕ CLI выключаем GNOME

```
sudo systemctl set-default multi-user.target
sudo reboot
```
##### На всех КРОМЕ ISP настраиваем DNS
```
nano /etc/resolv.conf
nameserver 192.168.1.2
```
#### На РОУТЕРАХ настроить ip_forward
```
echo net.ipv4.ip_forward=1 > /etc/sysctl.conf
```

| Маска подсети | Полная маска    | Сколько адресов вмещает |
| ------------- | --------------- | ----------------------- |
| /26           | 255.255.255.192 | 62                      |
| /27           | 255.255.255.224 | 32                      |
| /28           | 255.255.255.240 | 16                      |
| /29           | 255.255.255.248 | 8                       |


## Задание 1.  Произведите `базовую` настройку устройств
### Задание
- Настройте имена устройств согласно топологии. Используйте полное доменное имя
- На всех устройствах необходимо сконфигурировать IPv4:
	- IP-адрес должен быть из приватного диапазона, в случае, если сеть локальная, согласно RFC1918
	- Локальная сеть в сторону HQ-SRV(VLAN100) должна вмещать не более 32 адресов
	- Локальная сеть в сторону HQ-CLI(VLAN200) должна вмещать не более 16 адресов
	- Локальная сеть для управления(VLAN999) должна вмещать не более 8 адресов
	- Локальная сеть в сторону BR-SRV должна вмещать не более 16 адресов
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
address 192.168.1.1/27
vlan_raw_device ens19

#vlan 200
auto ens19.200
iface ens19.200 inet static
address 192.168.1.33/28
vlan_raw_device ens19

#vlan 999
auto ens19.999
iface ens19.999 inet static
address 192.168.1.49/29
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
address 192.168.2.1/28

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
address 192.168.2.2/28
gateway 192.168.2.1
```
##### HQ-SRV
```
allow-hotplug ens18.100
iface ens18.100 inet static
address 192.168.1.2/28
gateway 192.168.1.1
```
##### HQ-CLI
```
allow-hotplug ens18.200
iface ens18.200 inet dhcp
```

## Задание 2 `[NAT на ISP]`
### Задание
- Настройте адресацию на интерфейсах:
    - Интерфейс, подключенный к магистральному провайдеру, получает адрес по DHCP [Выполнено в задании 1]
    - Настройте маршруты по умолчанию там, где это необходимо
    - Интерфейс, к которому подключен HQ-RTR, подключен к сети 172.16.1.0/28 [Выполнено в задании 1]
    - Интерфейс, к которому подключен BR-RTR, подключен к сети 172.16.2.0/28 [Выполнено в задании 1]
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
- Создайте пользователя sshuser на серверах HQ-SRV и BR-SRV
    - Пароль пользователя sshuser с паролем P@ssw0rd
    - Идентификатор пользователя 2026
    - Пользователь sshuser должен иметь возможность запускать sudo без дополнительной аутентификации.
- Создайте пользователя net_admin на маршрутизаторах HQ-RTR и BR-RTR
    - Пароль пользователя net_admin с паролем P@ssw0rd
    - При настройке ОС на базе Linux, запускать sudo без ввода пароля 

### Решение
#### sshuser
**1.** Создаём sshuser следующими командами:
```
useradd sshuser -u 2026
passwd sshuser
P@ssw0rd
```
 
  **2.** После чего даем пользователю **`root`** права:
```yaml
usermod -aG sudo sshuser
```

**3.** Добавляем следующую строку в файле `/etc/sudoers`:
```
sshuser    ALL=(ALL:ALL) NOPASSWD: ALL
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
net_admin	ALL=(ALL:ALL) NOPASSWD: ALL
```

## Задание 4 `[VLAN]`
### Задание
#### Настройте на интерфейсе `HQ-RTR` в сторону офиса `HQ` виртуальный коммутатор
- Сервер HQ-SRV должен находиться в ID VLAN 100
- Клиент HQ-CLI в ID VLAN 200
- Создайте подсеть управления с ID VLAN 999
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
apt install openssh-server -y
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
sudo apt install frr -y
```

**2.** В конфигурационном файле `/etc/frr/daemons` необходимо активировать выбранный протокол `OSPF` для дальнейшей реализации его настройки:
```
nano /etc/frr/daemons

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
  network 192.168.1.0/27 area 1
  network 192.168.1.33/28 area 2
  network 192.168.1.48/29 area 3
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
- `network 192.168.2.0/27 area 4`
- `network 172.16.0.0/30 area 0`
    
```
conf t
router ospf
  passive-interface default
  router-id 2.2.2.2
  network 192.168.2.0/27 area 4
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
apt install iptables iptables-persistent –y
iptables –t nat –A POSTROUTING –s 192.168.1.0/27 –o ens18 –j MASQUERADE
iptables –t nat –A POSTROUTING –s 192.168.1.33/28 –o ens18 –j MASQUERADE
iptables –t nat –A POSTROUTING –s 192.168.1.48/29 –o ens18 –j MASQUERADE
netfilter-persistent save
systemctl restart netfilter-persistent  
```

#### Настройка динамической сетевой трансляции на `BR-RTR`
```
apt install iptables iptables-persistent –y
iptables –t nat –A POSTROUTING –s 192.168.2.0/27 –o ens18 –j MASQUERADE
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
authotitative;
default-lease-time 600;
max-lease-time 7200;
option domain-name "au-team.irpo";

subnet 192.168.1.32 netmask 255.255.255.240 {
  range 192.168.1.34 192.168.1.46;
  option domain-name-servers 192.168.1.2;
  option domain-name "au-team.irpo";
  option routers 192.168.1.33;
  option broadcast-address 192.168.1.47
  default-lease-time 600;
  max-lease-time 7200;
}
```

  **3.** После чего переходим в конфигурацию файла `isc-dhcp-server` и меняем ее добалвяя данный текст:
```
nano /etc/default/isc-dhcp-server

INTERFACESv4="ens19.200"
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

|            |                     |       |
| ---------- | ------------------- | ----- |
| Устройство | Запись              | Тип   |
| HQ-RTR     | hq-rtr.au-team.irpo | A,PTR |
| BR-RTR     | br-rtr.au-team.irpo | A     |
| HQ-SRV     | hq-srv.au-team.irpo | A,PTR |
| HQ-CLI     | hq-cli.au-team.irpo | A,PTR |
| BR-SRV     | br-srv.au-team.irpo | A     |
| HQ-RTR     | moodle.au-team.irpo | CNAME |
| BR-RTR     | wiki.au-team.irpo   | CNAME |

### Решение
#### HQ-SRV 
**1.** Для работы с **DNS** требуется установить **`bind`** и доп. пакет командой:
```
apt install bind9 bind9-utils
```

  **2.** Далее необходимо сконфигурировать файл **`named.conf.options`** таким образом:
```
nano /etc/bind/named.conf.options
```

```
forwarders {
		8.8.8.8;
		8.8.4.4;
};

allow-recursion {
		127.0.0.1;
		192.168.1.0/27;
		192.168.1.32/28;
		192.168.1.48/29;
		192.168.2.0/28;
};

allow-query {
		127.0.0.1;
		192.168.1.0/27;
		192.168.1.32/28;
		192.168.1.48/29;
		192.168.2.0/28;
};

listen-on {
		127.0.0.1;
		192.168.1.2;
};

dnssec-validation no;

listen-on-v6 { none; };
```

**3.** После чего требуется прописать в **`/etc/bind/named.conf.local`**:
```
zone "au-team.irpo" {
	  type master;
	  file "/var/lib/bind/db.au-team.irpo";
};

zone "1.168.192.in-addr.arpa" {
	  type master;
	  file "/var/lib/bind/db.1.168.192";
};

zone "2.168.192.in-addr.arpa" {
	  type master;
	  file "/var/lib/bind/db.2.168.192";
};
```

**4.** После чего **создаем файлы** командами:
```
cp /etc/bind/db.empty /var/lib/bind/db.au-team.irpo
cp /etc/bind/db.empty /var/lib/bind/db.1.168.192
cp /etc/bind/db.empty /var/lib/bind/db.2.168.192
```

**5.** ```nano /var/lib/bind/db.au-team.irpo```:

```
$TTL	604800
@	IN	SOA	hq-srv.au-team.irpo. root.au-team.irpo. (
			3		
			604800		
			86400		
			2419200	
			604800 
			)	
;
@	IN	NS	hq-srv.au-team.irpo.
hq-srv	IN	A	192.168.1.2
br-rtr	IN	A	192.168.2.1
br-srv	IN	A	192.168.2.2
hq-cli	IN	A	192.168.1.34
hq-rtr	IN	A	192.168.1.1
docker	IN	A	172.16.1.2
web	IN	A	172.16.2.2
```

**6.** ```nano /var/lib/bind/db.1.168.192```:

```
$TTL	604800
@	IN	SOA	hq-srv.au-team.irpo. root.au-team.irpo. (
			2		; Serial
			604800		; Refresh
			86400		; Retry
			2419200		; Expire
			604800
			)	; Negative Cache TTL
;
@	IN	NS	hq-srv.au-team.irpo.
1	IN	PTR	hq-rtr.au-team.irpo.
2	IN	PTR	hq-srv.au-team.irpo.
3	IN	PTR	hq-cli.au-team.irpo.
```

**6.** ```nano /etc/bind/2.168.192```:

```
$TTL	604800
@	IN	SOA	hq-srv.au-team.irpo. root.hq-srv.au-team.irpo. (
			1		; Serial
			604800		; Refresh
			86400		; Retry
			2419200		; Expire
			604800 
			)	; Negative Cache TTL
;
@	IN	NS	hq-srv.au-team.irpo.
1	IN	PTR	br-rtr.au-team.irpo.
2	IN	PTR	br-srv.au-team.irpo.
```

**7.** А также перезапускаем **`bind`** командой:
```
systemctl restart named bind9
```

**8.** На всех устройствах локальной сети необходимо указать в конфигурационном файле `resolv.conf`:
```
nameserver 192.168.1.2
search au-team.irpo
```

## Задание 11 `[ВРЕМЯ/ДАТА]`
### Задание
Настройте часовой пояс на всех устройствах, согласно месту проведения экзамена
### Решение
На Linux настраивается часовой пояс командой:
```
timedatectl set-timezone Europe/Moscow
```
