# Методичка к демоэкзамену

Данное руководство содержит пошаговые инструкции по настройке сетевого оборудования и серверов для демонстрационного экзамена.  
Все команды выполняются от **root** или с использованием `sudo`.

---

## Общие действия на всех устройствах

### Отключение systemd-resolved
```bash
systemctl disable --now systemd-resolved
unlink /etc/resolv.conf
```

> [!CAUTION]
> интерфейсы имеют другие имена (ens224, ens256, ens192). Везде подставляйте их, а не примеры из методички.

---

## Устройство ISP

### Имя устройства
```bash
hostnamectl set-hostname isp.au-team.irpo; exec bash
```
![Screenshot of hostname ISP.](assets/Снимок экрана 2026-02-09 142228.png)
### Закомментировать загрузку (отключение автоматической загрузки репозиториев)
```bash
nano /etc/apt/sources.list
```

Закомментируйте все строки, кроме необходимых (или приведите к нужному виду согласно заданию).

### Настройка сетевых адресов
```bash
nano /etc/network/interfaces
```
Приведите файл к виду (замените `enp0s8`/`enp0s9` на актуальные имена интерфейсов):
```ini
auto enp0s8
iface enp0s8 inet static
address 172.16.1.1
netmask 255.255.255.240
gateway 172.16.1.2

auto enp0s9
iface enp0s9 inet static
address 172.16.2.1
netmask 255.255.255.240
gateway 172.16.2.2
```
Перезапустим сеть:
```bash
systemctl restart networking
```

### Переадресация (NAT)
Раскомментируйте строку `net.ipv4.ip_forward=1` в файле:
```bash
nano /etc/sysctl.conf
```
Примените изменения:
```bash
sysctl -p
```
Установите iptables и настройте NAT:
```bash
apt install iptables iptables-persistent
iptables -t nat -A POSTROUTING -s 172.16.1.0/28 -o enp0s3 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 172.16.2.0/28 -o enp0s3 -j MASQUERADE
iptables-save > /etc/iptables/rules.v4
```
> ⚠️ **Важно:** интерфейс `enp0s3` может иметь другое имя (например, `ens192`). Подставьте своё.

### Настройка DNS
```bash
nano /etc/resolv.conf
```
Содержимое:
```
nameserver 1.1.1.1
```
Перезагрузите устройство:
```bash
reboot
```

---

## Устройство HQ-RTR

### Имя устройства
```bash
hostnamectl set-hostname hq-rtr.au-team.irpo; exec bash
```

### Закомментировать загрузку
```bash
nano /etc/apt/sources.list
```

### Настройка VLAN и интерфейсов
Установите поддержку VLAN:
```bash
apt install vlan
modprobe 8021q
echo 8021q >> /etc/modules
```
Отредактируйте `/etc/network/interfaces`:
```bash
nano /etc/network/interfaces
```
Пример содержания:
```ini
auto enp0s3
iface enp0s3 inet static
address 172.16.1.2
netmask 255.255.255.240
gateway 172.16.1.1

auto enp0s8
iface enp0s8 inet static
address 192.168.1.1
netmask 255.255.255.192

auto enp0s8:1
iface enp0s8:1 inet static
address 192.168.2.1
netmask 255.255.255.192

auto enp0s8.100
iface enp0s8.100 inet static
address 192.168.1.3
netmask 255.255.255.192
vlan_raw_device enp0s8

auto enp0s8.200
iface enp0s8.200 inet static
address 192.168.2.3
netmask 255.255.255.240
vlan_raw_device enp0s8:1

auto tun1
iface tun1 inet tunnel
address 10.10.0.1
netmask 255.255.255.252
mode gre
local 172.16.1.2
endpoint 172.16.2.2
ttl 64
```
Перезапустите сеть:
```bash
systemctl restart networking
```

### Добавление пользователя
```bash
adduser net_admin
# Пароль: P@ssw0rd
```
Выдайте привилегии через `visudo`:
```bash
visudo
```
Добавьте строку:
```
net_admin ALL=(ALL:ALL) ALL
```

### Переадресация (NAT)
Раскомментируйте `net.ipv4.ip_forward=1` в `/etc/sysctl.conf` и примените:
```bash
sysctl -p
```
Установите iptables:
```bash
apt install iptables iptables-persistent
iptables -t nat -A POSTROUTING -s 192.168.1.0/26 -o enp0s3 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.2.0/28 -o enp0s3 -j MASQUERADE
iptables-save > /etc/iptables/rules.v4
```
> ⚠️ **Важно:** замените `enp0s3` на актуальное имя интерфейса (например, `ens192`).

### Настройка DNS
```bash
nano /etc/resolv.conf
```
```
nameserver 1.1.1.1
```

### Настройка GRE туннеля
Добавьте модуль `ip_gre` в автозагрузку:
```bash
echo "ip_gre" >> /etc/modules
```
Перезапустите сеть:
```bash
systemctl restart networking
```

### Динамическая маршрутизация (OSPF)
Установите FRRouting:
```bash
apt install frr
```
Включите демон OSPF в `/etc/frr/daemons`:
```
ospfd=yes
```
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

### DHCP-сервер на HQ-RTR
Установите DHCP-сервер:
```bash
apt install isc-dhcp-server
```
Укажите интерфейс в `/etc/default/isc-dhcp-server`:
```ini
INTERFACESv4="enp0s8:1"   # замените на свой интерфейс, например ens224
```
Настройте пул в `/etc/dhcp/dhcpd.conf`:
```bash
nano /etc/dhcp/dhcpd.conf
```
Добавьте:
```
subnet 192.168.2.0 netmask 255.255.255.240 {
    range 192.168.2.4 192.168.2.14;
    option domain-name-servers 192.168.1.2;
    option domain-name "au-team.irpo";
    option routers 192.168.2.1;
    default-lease-time 600;
    max-lease-time 7200;
}
```
Перезапустите службу:
```bash
systemctl restart isc-dhcp-server
```

Перезагрузите устройство:
```bash
reboot
```

---

## Устройство BR-RTR

### Имя устройства
```bash
hostnamectl set-hostname br-rtr.au-team.irpo; exec bash
```

### Закомментировать загрузку
```bash
nano /etc/apt/sources.list
```

### Настройка адресов
```bash
nano /etc/network/interfaces
```
Пример:
```ini
auto enp0s3
iface enp0s3 inet static
address 172.16.2.2
netmask 255.255.255.240
gateway 172.16.2.1

auto enp0s8
iface enp0s8 inet static
address 192.168.4.1
netmask 255.255.255.240

auto tun1
iface tun1 inet tunnel
address 10.10.0.2
netmask 255.255.255.252
mode gre
local 172.16.2.2
endpoint 172.16.1.2
ttl 64
```
> ⚠️ **Важно:** подставьте свои имена интерфейсов (ens224, ens256 и т.д.)

Перезапустите сеть:
```bash
systemctl restart networking
```

### Добавление пользователя
```bash
adduser net_admin
# Пароль: P@ssw0rd
visudo
# Добавьте: net_admin ALL=(ALL:ALL) ALL
```

### Переадресация (NAT)
Раскомментируйте `net.ipv4.ip_forward=1` в `/etc/sysctl.conf`:
```bash
sysctl -p
```
Установите iptables:
```bash
apt install iptables iptables-persistent
iptables -t nat -A POSTROUTING -s 192.168.4.0/28 -o enp0s3 -j MASQUERADE
iptables-save > /etc/iptables/rules.v4
```
> ⚠️ **Важно:** замените `enp0s3` на актуальное имя (ens192).

### Настройка DNS
```bash
nano /etc/resolv.conf
```
```
nameserver 1.1.1.1
```

### Настройка GRE туннеля
```bash
echo "ip_gre" >> /etc/modules
systemctl restart networking
```

### Динамическая маршрутизация (OSPF)
```bash
apt install frr
```
В `/etc/frr/daemons` включите `ospfd=yes`.
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

Перезагрузите устройство:
```bash
reboot
```

---

## Устройство HQ-CLI

### Имя устройства
```bash
hostnamectl set-hostname hq-cli.au-team.irpo; exec bash
```

### Закомментировать загрузку
```bash
nano /etc/apt/sources.list
```

### Настройка адресов (DHCP)
```bash
nano /etc/network/interfaces
```
```ini
allow-hotplug enp0s3
iface enp0s3 inet dhcp
```
> ⚠️ **Важно:** замените `enp0s3` на актуальное имя (ens224, ens256).

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

---

## Устройство HQ-SRV

### Имя устройства
```bash
hostnamectl set-hostname hq-srv.au-team.irpo; exec bash
```

### Закомментировать загрузку
```bash
nano /etc/apt/sources.list
```

### Настройка адресов
```bash
nano /etc/network/interfaces
```
```ini
auto enp0s3
iface enp0s3 inet static
address 192.168.1.2
netmask 255.255.255.192
gateway 192.168.1.1
```
> ⚠️ **Важно:** замените `enp0s3` на актуальное имя (ens192).

Перезапустите сеть:
```bash
systemctl restart networking
```

### Добавление пользователя
```bash
adduser sshuser -u 2026
# Пароль: P@ssw0rd
visudo
# Добавьте: sshuser ALL=(ALL:ALL) ALL
```

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
Создайте баннер:
```bash
nano /etc/ssh-banner
```
```
Authorized access only
```
Перезапустите SSH:
```bash
systemctl restart sshd
```

### Настройка DNS-сервера (BIND9)
Установите BIND:
```bash
apt install bind9 dnsutils
```

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

#### 7. Проверка работы DNS
```bash
dig -x 192.168.1.1 @192.168.1.2
ss -tulpn | grep :53
```
С клиента (HQ-CLI) выполните:
```bash
nslookup 192.168.1.2
```

---

## Устройство BR-SRV

### Имя устройства
```bash
hostnamectl set-hostname br-srv.au-team.irpo; exec bash
```

### Закомментировать загрузку
```bash
nano /etc/apt/sources.list
```

### Настройка адресов
```bash
nano /etc/network/interfaces
```
```ini
auto enp0s8
iface enp0s8 inet static
address 192.168.4.2
netmask 255.255.255.240
gateway 192.168.4.1
```
> ⚠️ **Важно:** замените `enp0s8` на актуальное имя (ens192).

Перезапустите сеть:
```bash
systemctl restart networking
```

### Добавление пользователя
```bash
adduser sshuser -u 2026
# Пароль: P@ssw0rd
visudo
# Добавьте: sshuser ALL=(ALL:ALL) ALL
```

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
Создайте баннер:
```bash
nano /etc/ssh-banner
```
```
Authorized access only
```
Перезапустите SSH:
```bash
systemctl restart sshd
```

---

## Заключение

После выполнения всех шагов проверьте связность между устройствами, работу маршрутизации, DHCP и DNS.  
При возникновении ошибок сверьте имена интерфейсов и содержимое конфигурационных файлов с техническим заданием.
