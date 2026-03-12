# Методичка к демоэкзамену

Данное руководство содержит пошаговые инструкции по настройке сетевого оборудования и серверов для демонстрационного экзамена.  
Все команды выполняются от **root** или с использованием `sudo`.

## <p align="center"><b>МОДУЛЬ 1</b></p>

## <details><summary><p align="center"><b>Устройство ISP</p></summary>
<p>

### Имя устройства
```bash
hostnamectl set-hostname isp.au-team.irpo; exec bash
```
![Screenshot](assets/1.png)

### Закомментировать загрузку
```bash
nano /etc/apt/sources.list
```
![Screenshot](assets/2.png)

### Настройка сетевых адресов
```bash
nano /etc/network/interfaces
```
Приведите файл к виду (на скриншоте `enp0s8`/`enp0s9` заменены на `ens224`/`ens256`):
```ini
allow-hotplug ens224
iface ens224 inet static
address 172.16.1.1
netmask 255.255.255.240
gateway 172.16.1.2

allow-hotplug ens256
iface ens256 inet static
address 172.16.2.1
netmask 255.255.255.240
gateway 172.16.2.2
```
![Screenshot](assets/3.png)

> [!NOTE]
> Перед началом следует обновить устройство командой
> ```bash
>apt-get update -y
> ```

Перезапустим сеть:
```bash
systemctl restart networking
```

### Переадресация (NAT)
Вставляем строку `net.ipv4.ip_forward=1`, читаем файл и применяем в системе:
```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf; cat /etc/sysctl.conf; sysctl -p
```
![Screenshot](assets/6.png)

Установите iptables и настройте NAT:
```bash
apt install iptables iptables-persistent -y
iptables -t nat -A POSTROUTING -s 172.16.1.0/28 -o ens192 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 172.16.2.0/28 -o ens192 -j MASQUERADE
iptables-save >> /etc/iptables/rules.v4
```
![Screenshot](assets/7.png)

### Временная настройка DNS серверов 
```bash
nano /etc/resolv.conf
```
Вставляем временное содержимое:
```
nameserver 1.1.1.1
```
![Screenshot](assets/8.png)

---
</p>
</details>
## Устройство HQ-RTR

### Имя устройства
```bash
hostnamectl set-hostname hq-rtr.au-team.irpo; exec bash
```
![Screenshot](assets/5.png)

### Закомментировать загрузку
```bash
nano /etc/apt/sources.list
```
![Screenshot](assets/2.png)

### Настройка VLAN и интерфейсов
Отредактируйте `/etc/network/interfaces`:
```bash
nano /etc/network/interfaces
```
Пример содержания:
```ini
allow-hotplug ens192
iface ens192 inet static
address 172.16.1.2
netmask 255.255.255.240
gateway 172.16.1.1

allow-hotplug ens224
iface ens224 inet static
address 192.168.1.1
netmask 255.255.255.224
```
> [!NOTE]
> Писать именно до этого момента, если заполните все, конфиг будет ругаться на то что нет VLAN, и не сможет скачать VLAN :)

### Временная настройка DNS серверов 
```bash
nano /etc/resolv.conf
```
Временное содержимое:
```
nameserver 1.1.1.1
```
![Screenshot](assets/8.png)

> [!NOTE]
> Перед началом следует обновить устройство командой
> ```bash
>apt-get update -y
> ```

Установите VLAN:
```bash
apt install vlan -y
```
![Screenshot](assets/9.png)

Дайте поддержку VLAN
```bash
modprobe 8021q
echo 8021q >> /etc/modules
```
![Screenshot](assets/10.png)

Отредактируйте `/etc/network/interfaces`:
```bash
nano /etc/network/interfaces
```
Продолжите заполнять содержимое:
```bash
allow-hotplug ens224:1
iface ens224:1 inet static
address 192.168.2.1
netmask 255.255.255.224

allow-hotplug ens224.100
iface ens224.100 inet static
address 192.168.1.3
netmask 255.255.255.224
vlan_raw_device ens224

allow-hotplug ens224.200
iface ens224.200 inet static
address 192.168.2.3
netmask 255.255.255.224
vlan_raw_device ens224:1

allow-hotplug tun1
iface tun1 inet tunnel
address 10.10.0.1
netmask 255.255.255.252
mode gre
local 172.16.1.2
endpoint 172.16.2.2
ttl 64
```
![Screenshot](assets/11.png)

Перезапустите сеть:
```bash
systemctl restart networking
```

### Добавление пользователя
```bash
adduser net_admin
# Пароль: P@ssw0rd
```
![Screenshot](assets/23.png)

Выдайте привилегии через `visudo`:
```bash
visudo
```
Добавьте строку:
```
net_admin ALL=(ALL:ALL) ALL
```
![Screenshot](assets/25.png)

### Переадресация (NAT)
Вставляем строку `net.ipv4.ip_forward=1`, читаем файл и применяем в системе:
```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf; cat /etc/sysctl.conf; sysctl -p
```
![Screenshot](assets/6.png)

Установите iptables:
```bash
apt install iptables iptables-persistent -y
iptables -t nat -A POSTROUTING -s 192.168.1.0/27 -o ens192 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.2.0/27 -o ens192 -j MASQUERADE
iptables-save >> /etc/iptables/rules.v4
```
![Screenshot](assets/28.png)

### Временная настройка DNS серверов 
```bash
nano /etc/resolv.conf
```
Временное содержимое:
```
nameserver 1.1.1.1
```
![Screenshot](assets/8.png)

### Настройка GRE туннеля
Добавьте модуль `ip_gre` в автозагрузку:
```bash
echo "ip_gre" >> /etc/modules
```
![Screenshot](assets/33.png)

Перезапустите сеть:
```bash
systemctl restart networking
```

### Динамическая маршрутизация (OSPF)
Установите FRRouting:
```bash
apt install frr -y
```
![Screenshot](assets/34.png)

Включите демон OSPF в `/etc/frr/daemons`:
```
ospfd=yes
```
![Screenshot](assets/35.png)

Перезапустите FRR:
```bash
systemctl restart frr
```
Настройте OSPF через `vtysh`:
```bash
vtysh
conf t
router ospf
 passive-interface default
 network 192.168.1.0/26 area 0
 network 192.168.2.0/28 area 0
 network 10.10.0.0/30 area 0
 area 0 authentication
 exit
 interface tun1
  no ip ospf passive
  ip ospf authentication
  ip ospf authentication-key P@ssw0rd
 exit
 exit
 write memory
 exit
```
![Screenshot](assets/36.png)

>[!WARNING]
>Пока что я настраиваю frr и ip_gre т.к. нету связанности между srv-шками.
>тестовые команды
>```bash
>vtysh
>conf t
>router ospf
>router-id 10.10.0.1
>passive-interface default
>no passive-interface tun1
>network 192.168.1.0/27 area 0
>network 192.168.2.0/27 area 0
>network 10.10.0.0/30 area 0
>exit
>interface tun1
>ip ospf area 0
>ip ospf authentication
>ip ospf authentication-key P@ssw0rd
>exit
>exit
>write memory
>exit
>```
>`vtysh -c "show ip ospf neighbor"` соседи OSPF
>
>`vtysh -c "show ip route ospf"` таблица маршрутизации
>
>`vtysh -c "show running-config"` проверка конфига

### DHCP-сервер на HQ-RTR
Установите DHCP-сервер:
```bash
apt install isc-dhcp-server -y
```
![Screenshot](assets/38.png)

Укажите интерфейс в конфиге для DHCP:
```bash
nano /etc/default/isc-dhcp-server
```
То что туда нужно вписать:
```ini
INTERFACESv4="ens224:1"
```
![Screenshot](assets/40.png)

Настройте пул в `/etc/dhcp/dhcpd.conf`:
```bash
nano /etc/dhcp/dhcpd.conf
```
Добавьте:
```
subnet 192.168.2.0 netmask 255.255.255.224 {
    range 192.168.2.4 192.168.2.19;
    option domain-name-servers 192.168.1.2;
    option domain-name "au-team.irpo";
    option routers 192.168.2.1;
    default-lease-time 600;
    max-lease-time 7200;
}
```
![Screenshot](assets/41.png)

Перезапустите службу:
```bash
systemctl restart isc-dhcp-server
```

---
---

## Устройство BR-RTR

### Имя устройства
```bash
hostnamectl set-hostname br-rtr.au-team.irpo; exec bash
```
![Screenshot](assets/4.png)

### Закомментировать загрузку
```bash
nano /etc/apt/sources.list
```
![Screenshot](assets/2.png)

### Настройка адресов
```bash
nano /etc/network/interfaces
```
Пример:
```ini
allow-hotplug ens192
iface ens192 inet static
address 172.16.2.2
netmask 255.255.255.240
gateway 172.16.2.1

allow-hotplug ens224
iface ens224 inet static
address 192.168.4.1
netmask 255.255.255.240

allow-hotplug tun1
iface tun1 inet tunnel
address 10.10.0.2
netmask 255.255.255.252
mode gre
local 172.16.2.2
endpoint 172.16.1.2
ttl 64
```
![Screenshot](assets/16.png)

Перезапустите сеть:
```bash
systemctl restart networking
```

### Временная настройка DNS серверов 
```bash
nano /etc/resolv.conf
```
Временное содержимое:
```
nameserver 1.1.1.1
```
![Screenshot](assets/8.png)

### Добавление пользователя
```bash
adduser net_admin
# Пароль: P@ssw0rd
```
![Screenshot](assets/24.png)

Выдайте привилегии через `visudo`:
```bash
visudo
```
Добавьте строку:
```
net_admin ALL=(ALL:ALL) ALL
```
![Screenshot](assets/25.png)

### Переадресация (NAT)
Вставляем строку `net.ipv4.ip_forward=1`, читаем файл и применяем в системе:
```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf; cat /etc/sysctl.conf; sysctl -p
```
![Screenshot](assets/6.png)

Примените изменения:
```bash
sysctl -p
```
Установите iptables:
```bash
apt install iptables iptables-persistent -y
iptables -t nat -A POSTROUTING -s 192.168.4.0/28 -o ens192 -j MASQUERADE
iptables-save >> /etc/iptables/rules.v4
```
![Screenshot](assets/29.png)

Добавьте модуль `ip_gre` в автозагрузку:
```bash
echo "ip_gre" >> /etc/modules
```
![Screenshot](assets/33.png)

Перезапустите сеть:
```bash
systemctl restart networking
```

### Динамическая маршрутизация (OSPF)
Установите FRRouting:
```bash
apt install frr -y
```
![Screenshot](assets/34.png)

Включите демон OSPF в `/etc/frr/daemons`:
```
ospfd=yes
```
![Screenshot](assets/35.png)

Перезапустите FRR:
```bash
systemctl restart frr
```

Настройка через `vtysh`:
```bash
vtysh
conf t
router ospf
passive-interface default
network 192.168.4.0/28 area 0
network 10.10.0.0/30 area 0
area 0 authentication
exit
interface tun1
no ip ospf passive
ip ospf authentication
ip ospf authentication-key P@ssw0rd
exit
exit
write memory
exit
```
![Screenshot](assets/37.png)

>[!WARNING]
>Пока что я настраиваю frr и ip_gre т.к. нету связанности между srv-шками.
>тестовый код
>```bash
>vtysh
>conf t
>router ospf
>router-id 10.10.0.2
>passive-interface default
>no passive-interface tun1
>network 192.168.4.0/28 area 0
>network 10.10.0.0/30 area 0
>exit
>interface tun1
>ip ospf area 0
>ip ospf authentication
>ip ospf authentication-key P@ssw0rd
>exit
>exit
>write memory
>exit
>```
>`vtysh -c "show ip ospf neighbor"` соседи OSPF
>`vtysh -c "show ip route ospf"` таблица маршрутизации
>`vtysh -c "show running-config"` проверка конфига

---
---

## Устройство HQ-CLI

### Имя устройства
```bash
hostnamectl set-hostname hq-cli.au-team.irpo; exec bash
```
![Screenshot](assets/17.png)

### Закомментировать загрузку
```bash
nano /etc/apt/sources.list
```
![Screenshot](assets/2.png)

### Настройка адресов (DHCP)
```bash
nano /etc/network/interfaces
```
```ini
allow-hotplug ens192
iface ens192 inet dhcp
```
![Screenshot](assets/42.png)

Перезапустите сеть:
```bash
systemctl restart networking
```

### Настройка DNS
```bash
nano /etc/resolv.conf
```
```
search au-team.irpo
nameserver 192.168.1.2
```
![Screenshot](assets/54.png)

> [!CAUTION]
> Также следует обновить пакеты командой
> ```bash
>apt-get update -y
> ```

---
---

## Устройство HQ-SRV

### Имя устройства
```bash
hostnamectl set-hostname hq-srv.au-team.irpo; exec bash
```
![Screenshot](assets/12.png)

### Закомментировать загрузку
```bash
nano /etc/apt/sources.list
```
![Screenshot](assets/2.png)

### Временная настройка DNS серверов 
```bash
nano /etc/resolv.conf
```
Временное содержимое:
```
nameserver 1.1.1.1
```
![Screenshot](assets/8.png)

### Настройка адресов
```bash
nano /etc/network/interfaces
```
```ini
allow-hotplug ens192
iface ens192 inet static
address 192.168.1.2
netmask 255.255.255.224
gateway 192.168.1.1
```
![Screenshot](assets/13.png)

Перезапустите сеть:
```bash
systemctl restart networking
```

> [!CAUTION]
> Перед началом следует обновить устройство командой
> ```bash
>apt-get update -y
> ```

### Добавление пользователя
```bash
adduser sshuser -u 2026
# Пароль: P@ssw0rd
```
![Screenshot](assets/27.png)

Выдайте привилегии через `visudo`:
```bash
visudo
```
Добавьте строку:
```
sshuser ALL=(ALL:ALL) ALL
```
![Screenshot](assets/21.png)

### Настройка SSH
```bash
nano /etc/ssh/sshd_config
```
Внесите изменения:
```
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/ssh-banner
```
![Screenshot](assets/30.png)

Создайте баннер:
```bash
nano /etc/ssh-banner
```
```
----------------------
Authorized access only
----------------------
```
![Screenshot](assets/32.png)

Перезапустите SSH:
```bash
systemctl restart sshd
```

### Настройка DNS-сервера (BIND9)
Установите BIND:
```bash
apt install bind9 dnsutils -y
```
![Screenshot](assets/44.png)

#### 1. Файл `/etc/bind/named.conf.options`
```bash
nano /etc/bind/named.conf.options
```
```nginx
listen-on port 53 { any; };
listen-on-v6 port 53 { any; };
directory "/var/cache/bind";
allow-query { any; };
allow-recursion { any; };
forwarders { 77.88.8.8; 1.1.1.1; };
forward only;
dnssec-validation no;
```
![Screenshot](assets/56.png)

#### 2. Файл `/etc/bind/named.conf.default-zones`
Добавьте в конец:
```bash
nano /etc/bind/named.conf.default-zones
```
```nginx
zone "au-team.irpo" {
    type master;
    file "/etc/bind/au-team.irpo";
    allow-query { any; };
    allow-transfer { any; };
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/au-team.irpo_obr";
};

zone "2.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/au-team.irpo_hqobr";
};
```
![Screenshot](assets/49.png)

#### 3. Прямая зона `au-team.irpo`
Скопируйте шаблон:
```bash
cp /etc/bind/db.local /etc/bind/au-team.irpo
nano /etc/bind/au-team.irpo
```
Приведите к виду:
```dns
$TTL    604800
@       IN      SOA     hq-srv.au-team.irpo. root.au-team.irpo. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      hq-srv.au-team.irpo.
hq-srv  IN      A       192.168.1.2
hq-rtr  IN      A       192.168.1.1
hq-cli  IN      A       192.168.2.2
br-rtr  IN      A       192.168.4.1
br-srv  IN      A       192.168.4.2
moodle  IN      CNAME   hq-rtr.au-team.irpo.
wiki    IN      CNAME   hq-rtr.au-team.irpo.
```
![Screenshot](assets/50.png)

#### 4. Обратная зона для 192.168.1.0/26
```bash
cp /etc/bind/db.127 /etc/bind/au-team.irpo_obr
nano /etc/bind/au-team.irpo_obr
```
```dns
$TTL    604800
@       IN      SOA     hq-srv.au-team.irpo. root.au-team.irpo. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      hq-srv.au-team.irpo.
1       IN      PTR     hq-rtr.au-team.irpo.
2       IN      PTR     hq-srv.au-team.irpo.
```
![Screenshot](assets/51.png)

#### 5. Обратная зона для 192.168.2.0/28
Скопируйте предыдущую и отредактируйте:
```bash
cp /etc/bind/au-team.irpo_obr /etc/bind/au-team.irpo_hqobr
nano /etc/bind/au-team.irpo_hqobr
```
```dns
$TTL    604800
@       IN      SOA     hq-srv.au-team.irpo. root.au-team.irpo. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      hq-srv.au-team.irpo.
2       IN      PTR     hq-cli.au-team.irpo.
```
![Screenshot](assets/53.png)

#### 6. Проверка конфигурации
```bash
named-checkconf
named-checkzone au-team.irpo /etc/bind/au-team.irpo
named-checkzone 1.168.192.in-addr.arpa /etc/bind/au-team.irpo_obr
named-checkzone 2.168.192.in-addr.arpa /etc/bind/au-team.irpo_hqobr
```
Ожидайте вывод **OK**.

Перезапустите BIND:
```bash
systemctl restart bind9
```

>[!IMPORTANT]
>**На DNS не грешить, конфигурация была проверена много раз с использованием нескольких средств виртуализации, если что-либо идет не так, виновата другая часть конфигурации**

#### 7. Проверка работы DNS (проверка на hq-srv, hq-rtr)
Переходим в resolv.conf:
```bash
nano /etc/resolv.conf
```
Вставляем в конфиг:
```bash
search au-team.irpo
nameserver 192.168.1.2
```
```bash
dig -x 192.168.1.1 @192.168.1.2
```
С клиента (HQ-CLI) выполните:
```bash
nslookup 192.168.1.2
```
![Screenshot](assets/55.png)

---

## Устройство BR-SRV

### Имя устройства
```bash
hostnamectl set-hostname br-srv.au-team.irpo; exec bash
```
![Screenshot](assets/4.png)

### Закомментировать загрузку
```bash
nano /etc/apt/sources.list
```
### Временная настройка DNS серверов 
```bash
nano /etc/resolv.conf
```
Временное содержимое:
```
nameserver 1.1.1.1
```
![Screenshot](assets/8.png)

### Настройка адресов
```bash
nano /etc/network/interfaces
```
```ini
allow-hotplug ens192
iface ens192 inet static
address 192.168.4.2
netmask 255.255.255.240
gateway 192.168.4.1
```
![Screenshot](assets/15.png)

Перезапустите сеть:
```bash
systemctl restart networking
```

> [!CAUTION]
> Перед началом следует обновить устройство командой
> ```bash
>apt-get update -y
> ```

### Добавление пользователя
```bash
adduser sshuser -u 2026
# Пароль: P@ssw0rd
```
![Screenshot](assets/19.png)

Выдайте привилегии через `visudo`:
```bash
visudo
```
Добавьте строку:
```
sshuser ALL=(ALL:ALL) ALL
```
![Screenshot](assets/21.png)


### Настройка SSH
```bash
nano /etc/ssh/sshd_config
```
```
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/ssh-banner
```
![Screenshot](assets/30.png)

Создайте баннер:
```bash
nano /etc/ssh-banner
```
```
Authorized access only
```
![Screenshot](assets/32.png)

Перезапустите SSH:
```bash
systemctl restart sshd
```

---

> [!NOTE]
> Самое главное, после всего этого, на HQ-RTR, HQ-SRV, HQ-CLI, BR-RTR, BR-SRV выставить новый DNS
```
search au-team.irpo
nameserver 192.168.1.2
```
![Screenshot](assets/54.png)

timedatectl set-timezone Asia/Tomsk
нужно настроить через dnsmasq

>[!IMPORTANT]
>Доп команды: разрыв resolv.conf с остальным 
>```bash
>unlink /etc/resolv.conf
>```
>Можно посмотрть сохраненные правила iptables через команду:
>```bash
>iptables -L -t nat
>```
>Если вы по какой то причине перезагрузили устройство, не забывайте про команду
>```bash
>sysctl -p
>```
>чтобы применить переадресацию NAT
>
>Иногда виртуалки зависают так как вставка команды идет построчно, это нормально
>
>Более быстрый способ создания пользователя (экономия = 2 секунды)
>```bash
>useradd -p P@ssw0rd net_admin
>```
