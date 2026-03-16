# Методичка к демоэкзамену

Данное руководство содержит пошаговые инструкции по настройке сетевого оборудования и серверов для демонстрационного экзамена.  
Все команды выполняются от **root** или с использованием `sudo`.

| Имя устройства | Интерфейс | IP-адрес | Шлюз | Сеть |
| --- | --- | --- | --- | --- |
| ISP | ens192 | DHCP | - | INTERNET |
|  | ens224 | 172.16.1.1/28 | - | ISP-HQ |
|  | ens256 | 172.16.2.1/28 | - | ISP-BR |
| HQ-RTR | ens192 | 172.16.1.2/28 | 172.16.1.1 | ISP-HQ |
|  | ens224(VLAN100) | 192.168.1.1/27 | - | SRV-NET |
|  | ens224(VLAN200) | 192.168.2.1/27 | - | CLI-NET |
| HQ-SRV | ens192 | 192.168.1.2/27 | 192.168.1.1 | SRV-NET |
| HQ-CLI | ens192 | DHCP | 192.168.2.1 | CLI-NET |
| BR-RTR | ens192 | 172.16.2.2/28 | 172.16.2.1 | ISP-BR |
|  | ens224 | 192.168.4.1/28 | - | BR-NET |
| BR-SRV | ens192 | 192.168.4.2/28 | 192.168.4.1 | BR-NET |

## <p align="center"><b>МОДУЛЬ 1</b></p>

## <p align="center"><b>Устройство ISP</p>

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

allow-hotplug ens256
iface ens256 inet static
address 172.16.2.1
netmask 255.255.255.240
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

### Настройка часового пояса
```bash
timedatectl set-timezone Asia/Tomsk
```
---
## <p align="center"><b>Устройство HQ-RTR</p>

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
router-id 10.10.0.1
passive-interface default
network 192.168.1.0/27 area 0
network 192.168.2.0/27 area 0
network 10.10.0.0/30 area 0
exit
interface tun1
no ip ospf passive
ip ospf area 0
ip ospf authentication
ip ospf authentication-key P@ssw0rd
exit
exit
write memory
exit
```
>`vtysh -c "show ip ospf neighbor"` соседи OSPF
>
>`vtysh -c "show ip route ospf"` таблица маршрутизации
>
>`vtysh -c "show running-config"` проверка конфига
![Screenshot](assets/36.png)

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
### Настройка часового пояса
```bash
timedatectl set-timezone Asia/Tomsk
```

---

## <p align="center"><b>Устройство BR-RTR</p>

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
>
>`vtysh -c "show ip route ospf"` таблица маршрутизации
>
>`vtysh -c "show running-config"` проверка конфига
![Screenshot](assets/37.png)

### Настройка часового пояса
```bash
timedatectl set-timezone Asia/Tomsk
```

---

## <p align="center"><b>Устройство HQ-CLI</p>

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

### Настройка часового пояса
```bash
timedatectl set-timezone Asia/Tomsk
```

---

## <p align="center"><b>Устройство HQ-SRV</p>

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
forwarders { 77.88.8.7; 1.1.1.1; };
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
                              3         ; Serial
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
docker  IN      A       172.16.2.1
web     IN      A       172.16.1.1
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

### Настройка часового пояса
```bash
timedatectl set-timezone Asia/Tomsk
```

---

## <p align="center"><b>Устройство BR-SRV</p>

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

### Настройка часового пояса
```bash
timedatectl set-timezone Asia/Tomsk
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
Зеркала Яндекса
```bash
deb http://mirror.yandex.ru/debian/ bookworm main
deb-src http://mirror.yandex.ru/debian/ bookworm main
deb http://mirror.yandex.ru/debian-security bookworm-security main contrib
deb-src http://mirror.yandex.ru/debian-security bookworm-security main contrib
deb http://mirror.yandex.ru/debian/ bookworm-updates main contrib
deb-src http://mirror.yandex.ru/debian/ bookworm-updates main contrib
```

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

---

## <p align="center"><b>МОДУЛЬ 2</b></p>

---

## <p align="center"><b>1. Samba AD DC на BR-SRV</b></p>

### Установка необходимых пакетов
```bash
apt install samba smbclient winbind krb5-config -y
```

При установке `krb5-config` будут запрошены параметры:
- **Default Kerberos realm**: `AU-TEAM.IRPO`
- **Kerberos servers**: `br-srv.au-team.irpo`
- **Administrative server**: `br-srv.au-team.irpo`

### Остановка служб и очистка конфигурации
```bash
systemctl stop samba-ad-dc smbd nmbd winbind 2>/dev/null
systemctl disable samba-ad-dc smbd nmbd winbind 2>/dev/null
mv /etc/samba/smb.conf /etc/samba/smb.conf.bak
```

### Развёртывание домена
```bash
samba-tool domain provision --use-rfc2307 --interactive
```
Параметры при установке:
- **Realm**: `AU-TEAM.IRPO`
- **Domain**: `AU-TEAM`
- **Server Role**: `dc`
- **DNS backend**: `SAMBA_INTERNAL`
- **DNS forwarder IP**: `192.168.1.2`
- **Administrator password**: `P@ssw0rd`

### Настройка Kerberos
```bash
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

### Запуск Samba AD DC
```bash
systemctl unmask samba-ad-dc
systemctl enable samba-ad-dc
systemctl start samba-ad-dc
```

### Проверка работы домена
```bash
samba-tool domain level show
smbclient -L localhost -N
```

### Создание пользователей и группы
Создайте группу `hq`:
```bash
samba-tool group add hq
```

Создайте 5 пользователей:
```bash
samba-tool user create hquser1 P@ssw0rd
samba-tool user create hquser2 P@ssw0rd
samba-tool user create hquser3 P@ssw0rd
samba-tool user create hquser4 P@ssw0rd
samba-tool user create hquser5 P@ssw0rd
```

Добавьте пользователей в группу `hq`:
```bash
samba-tool group addmembers hq hquser1,hquser2,hquser3,hquser4,hquser5
```

### Ограничение команд для группы hq
Создайте каталог и символические ссылки для разрешённых команд:
```bash
mkdir -p /usr/local/bin/restricted
ln -sf /usr/bin/cat /usr/local/bin/restricted/cat
ln -sf /usr/bin/grep /usr/local/bin/restricted/grep
ln -sf /usr/bin/id /usr/local/bin/restricted/id
```

Создайте скрипт `/usr/local/bin/restricted_shell.sh`:
```bash
nano /usr/local/bin/restricted_shell.sh
```
```bash
#!/bin/bash
export PATH="/usr/local/bin/restricted"
exec /bin/rbash
```
```bash
chmod +x /usr/local/bin/restricted_shell.sh
```

> [!IMPORTANT]
> После ввода пользователей в домен и входа на HQ-CLI, ограничение команд будет применяться через назначение этого скрипта как оболочки.
> На BR-SRV для каждого пользователя можно задать оболочку:
> ```bash
> samba-tool user setattr hquser1 loginShell /usr/local/bin/restricted_shell.sh
> samba-tool user setattr hquser2 loginShell /usr/local/bin/restricted_shell.sh
> samba-tool user setattr hquser3 loginShell /usr/local/bin/restricted_shell.sh
> samba-tool user setattr hquser4 loginShell /usr/local/bin/restricted_shell.sh
> samba-tool user setattr hquser5 loginShell /usr/local/bin/restricted_shell.sh
> ```
> Также скопируйте restricted_shell.sh и каталог /usr/local/bin/restricted на HQ-CLI.

### Ввод HQ-CLI в домен
На **HQ-CLI** установите необходимые пакеты:
```bash
apt install realmd sssd sssd-tools adcli packagekit samba-common-bin -y
```

Настройте DNS на HQ-CLI для использования BR-SRV:
```bash
nano /etc/resolv.conf
```
```
search au-team.irpo
nameserver 192.168.4.2
```

Введите машину в домен:
```bash
realm join AU-TEAM.IRPO -U Administrator
# Пароль: P@ssw0rd
```
>[!WARNING]
>Если на HQ-CLI во время присоединения к домену выдают ошибку, то стоит проверить точность времени на BR-SRV и HQ-CLI путем команды `date`, так как иногда происходят приколы с точностью даты и времени на виртуалках 
```bash
date --set "2026-03-16 10:55:36"
#Ручное выставление даты, формат: yyyy-mm-dd hh:mm:ss
dpkg-reconfigure tzdata
#Выбор автоматической даты (именно после предыдущей команды)
5
79
#Выбор Томска
```
Проверка:
```bash
realm list
id hquser1@au-team.irpo
```

Настройте автоматическое создание домашних каталогов:
```bash
nano /etc/pam.d/common-session
```
Добавьте строку:
```
session required pam_mkhomedir.so skel=/etc/skel umask=077
```

Проверьте вход под доменным пользователем:
```bash
su - hquser1@au-team.irpo
```

> [!NOTE]
> После ввода в домен верните DNS обратно:
> ```
> search au-team.irpo
> nameserver 192.168.1.2
> ```

---

## <p align="center"><b>2. RAID-массив на HQ-SRV</b></p>

### Установка mdadm
```bash
apt install mdadm -y
```

### Просмотр доступных дисков
```bash
lsblk
```
> На виртуалке два доп. диска по 1 ГБ. Предположим, что доступны диски `/dev/sdb` и `/dev/sdc`.

### Создание RAID 1 (зеркало)
```bash
mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
```
Подтвердите: `y`

### Проверка состояния RAID
```bash
cat /proc/mdstat
mdadm --detail /dev/md0
```

### Сохранение конфигурации
```bash
mdadm --detail --scan >> /etc/mdadm/mdadm.conf
update-initramfs -u
```

### Создание файловой системы ext4
```bash
mkfs.ext4 /dev/md0
```

### Монтирование в /raid
```bash
mkdir -p /raid
mount /dev/md0 /raid
```

Добавьте в `/etc/fstab` для автомонтирования:
```bash
echo '/dev/md0 /raid ext4 defaults 0 0' >> /etc/fstab
```

### Проверка
```bash
df -h /raid
```

---

## <p align="center"><b>3. NFS (сетевая файловая система) на HQ-SRV</b></p>

### Установка NFS-сервера на HQ-SRV
```bash
apt install nfs-kernel-server -y
```

### Создание каталога для NFS
```bash
mkdir -p /raid/nfs
chmod 777 /raid/nfs
```

### Настройка экспорта
```bash
nano /etc/exports
```
Добавьте:
```
/raid/nfs 192.168.2.0/27(rw,sync,no_subtree_check,no_root_squash)
```

### Применение и запуск
```bash
exportfs -a
systemctl restart nfs-kernel-server
systemctl enable nfs-kernel-server
```

### Настройка NFS-клиента на HQ-CLI
```bash
apt install nfs-common -y
```

```bash
mkdir -p /mnt/nfs
mount 192.168.1.2:/raid/nfs /mnt/nfs
```

Добавьте в `/etc/fstab`:
```bash
echo '192.168.1.2:/raid/nfs /mnt/nfs nfs defaults 0 0' >> /etc/fstab
```

### Проверка
На HQ-SRV создайте файл:
```bash
touch /raid/nfs/test_file
```

На HQ-CLI проверьте:
```bash
ls /mnt/nfs/
```

---

## <p align="center"><b>4. Настройка NTP (chrony) на ISP</b></p>

### Установка chrony на ISP
```bash
apt install chrony -y
```

### Настройка ISP как NTP-сервера
```bash
nano /etc/chrony/chrony.conf
```
Замените/добавьте:
```ini
local stratum 5
allow 172.16.0.0/28
allow 172.16.1.0/28
allow 172.16.2.0/28
allow 192.168.0.0/16
```

Перезапустите:
```bash
systemctl restart chrony
systemctl enable chrony
```

### Настройка NTP-клиентов

На **HQ-SRV**, **HQ-CLI**, **HQ-RTR** установите chrony и настройте:
```bash
apt install chrony -y
nano /etc/chrony/chrony.conf
```

Закомментируйте строку `pool` и добавьте:
```ini
server 172.16.1.1 iburst
```

На **BR-RTR** и **BR-SRV**:
```bash
apt install chrony -y
nano /etc/chrony/chrony.conf
```

Закомментируйте строку `pool` и добавьте:
```ini
server 172.16.2.1 iburst
```

Перезапустите на всех клиентах:
```bash
systemctl restart chrony
systemctl enable chrony
```

### Проверка синхронизации
```bash
chronyc sources
chronyc tracking
```

---

## <p align="center"><b>5. Ansible на BR-SRV</b></p>

### Установка Ansible
```bash
apt install ansible sshpass -y
```

### Настройка инвентаря
```bash
mkdir -p /etc/ansible
nano /etc/ansible/hosts
```
>У HQ-CLI скорее всего другой IPv4
```ini
[hq]
hq-srv ansible_host=192.168.1.2 ansible_user=sshuser ansible_password=P@ssw0rd ansible_port=2026
hq-rtr ansible_host=192.168.1.1 ansible_user=net_admin ansible_password=P@ssw0rd
hq-cli ansible_host=192.168.2.3 ansible_user=root ansible_password=P@ssw0rd

[br]
br-rtr ansible_host=192.168.4.1 ansible_user=net_admin ansible_password=P@ssw0rd
```
>[!NOTE]
>Если с HQ-CLI не приходит ответ вида **pong**, то `apt install openssh-server`

### Настройка ansible.cfg
```bash
nano /etc/ansible/ansible.cfg
```
```ini
[defaults]
inventory = /etc/ansible/hosts
host_key_checking = False
```

### Проверка подключения (ping - pong)
```bash
ansible all -m ping
```
Ожидаемый вывод: `pong` от каждого хоста.

> [!NOTE]
> Для хостов без нестандартного порта SSH порт по умолчанию 22.

---

## <p align="center"><b>6. Docker на BR-SRV</b></p>

### Установка Docker
```bash
apt install docker.io docker-compose -y
systemctl enable docker
systemctl start docker
```

### Просмотр доступных дисков
```bash
lsblk
```

### Загрузка образов из Additional.iso
> Подмонтируйте Additional.iso и загрузите образы:
```bash
mkdir -p /mnt/iso
mount /dev/cdrom /mnt/iso
docker load < /mnt/iso/docker/site_latest.tar
docker load < /mnt/iso/docker/mariadb_latest.tar
```

### Проверка загруженных образов
```bash
docker images # Этой командой можно узнать ТЕГ образа для yml файла
```

### Создание docker-compose.yml
```bash
mkdir -p /opt/testapp
nano /opt/testapp/docker-compose.yml
```
> Данные я и сам пока не знаю, но они будут дописаны, расположение данных `/mnt/iso/docker/readme.txt`
```yaml
version: '3'

services:
  testapp:
    image: site:latest
    container_name: testapp
    restart: always
    ports:
      - "8080:8000"
    environment:
      DB_HOST: "192.168.4.2"
      DB_PORT: "3306"
      DB_NAME: testdb
      DB_USER: test
      DB_PASS: P@ssw0rd
      DB_TYPE: maria
    depends_on:
      - db

  db:
    image: mariadb:10.11
    container_name: db
    ports:
      - "3306:3306"
    environment:
      MARIADB_DATABASE: testdb
      MARIADB_USER: test
      MARIADB_PASSWORD: P@ssw0rd
      MARIADB_ROOT_PASSWORD: P@ssw0rd
    restart: always
```

### Запуск контейнеров
```bash
cd /opt/testapp
docker-compose up -d
```

### Проверка
```bash
docker ps
apt install curl -y
curl http://localhost:8080
```

---

## <p align="center"><b>7. Apache + MariaDB + web на HQ-SRV</b></p>

### Установка Apache
```bash
apt install apache2 -y
systemctl enable apache2
systemctl start apache2
```

### Установка MariaDB
```bash
apt install mariadb-server mariadb-client -y
systemctl enable mariadb
systemctl start mariadb
```

### Установка PHP
```bash
apt install php libapache2-mod-php php-mysql -y
```

### Импорт базы данных
> Подмонтируйте Additional.iso:
```bash
mkdir -p /mnt/iso
mount /dev/cdrom /mnt/iso
```

Создайте базу данных и пользователя:
```bash
mysql -u root <<EOF
CREATE DATABASE webdb;
CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost';
FLUSH PRIVILEGES;
EOF
```

Импортируйте дамп:
```bash
mysql -u root webdb < /mnt/iso/web/dump.sql
```

### Размещение файлов сайта
Скопируйте `index.php` и каталог `images` из Additional.iso:
```bash
cp /mnt/iso/web/index.php /var/www/html/
cp -r /mnt/iso/web/images /var/www/html/
#Возможно за место директории images будет файл logo.png
chown -R www-data:www-data /var/www/html/
```

> [!NOTE]
> Убедитесь, что `index.php` содержит правильные параметры подключения к БД. При необходимости отредактируйте:
> ```bash
> nano /var/www/html/index.php
> ```
> Проверьте параметры: хост `localhost`, БД `webdb`, пользователь `web`, пароль `P@ssw0rd`.

Удалите стандартную страницу:
```bash
rm -f /var/www/html/index.html
```

Перезапустите Apache:
```bash
systemctl restart apache2
```

### Проверка
```bash
apt install curl -y
curl http://localhost
```

---

## <p align="center"><b>8. Проброс портов (iptables)</b></p>

### На BR-RTR: проброс портов 8080, 2026 на BR-SRV
```bash
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.4.2:8080
iptables -t nat -A PREROUTING -p tcp --dport 2026 -j DNAT --to-destination 192.168.4.2:2026
iptables -A FORWARD -p tcp -d 192.168.4.2 --dport 8080 -j ACCEPT
iptables -A FORWARD -p tcp -d 192.168.4.2 --dport 2026 -j ACCEPT
iptables-save > /etc/iptables/rules.v4
```

### На HQ-RTR: проброс порта 8080, 2026 на HQ-SRV
```bash
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.2:80
iptables -t nat -A PREROUTING -p tcp --dport 2026 -j DNAT --to-destination 192.168.1.2:2026
iptables -A FORWARD -p tcp -d 192.168.1.2 --dport 80 -j ACCEPT
iptables -A FORWARD -p tcp -d 192.168.1.2 --dport 2026 -j ACCEPT
iptables-save > /etc/iptables/rules.v4
```

---

## <p align="center"><b>9. Nginx reverse proxy на ISP</b></p>

### Установка Nginx
```bash
apt install nginx -y
```

### Настройка reverse proxy для web.au-team.irpo
```bash
nano /etc/nginx/sites-available/web.au-team.irpo
```
```nginx
server {
    listen 80;
    server_name web.au-team.irpo;

    location / {
        proxy_pass http://172.16.1.2:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Настройка reverse proxy для docker.au-team.irpo
```bash
nano /etc/nginx/sites-available/docker.au-team.irpo
```
```nginx
server {
    listen 80;
    server_name docker.au-team.irpo;

    location / {
        proxy_pass http://172.16.2.2:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Активация конфигураций
```bash
ln -s /etc/nginx/sites-available/web.au-team.irpo /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/docker.au-team.irpo /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default
nginx -t
systemctl restart nginx
systemctl enable nginx
```

---

## <p align="center"><b>10. Basic Auth на ISP (web.au-team.irpo)</b></p>

### Установка утилиты htpasswd
```bash
apt install apache2-utils -y
```

### Создание файла паролей
```bash
htpasswd -cb /etc/nginx/.htpasswd WEB P@ssw0rd
```

### Обновление конфигурации Nginx
```bash
nano /etc/nginx/sites-available/web.au-team.irpo
```
```nginx
server {
    listen 80;
    server_name web.au-team.irpo;

    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass http://172.16.1.2:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Перезапустите Nginx:
```bash
nginx -t
systemctl restart nginx
```

---

## <p align="center"><b>11. Проверка доступа с HQ-CLI</b></p>

С HQ-CLI откройте браузер и проверьте:

1. **web.au-team.irpo** — должен запросить логин/пароль (WEB / P@ssw0rd), после чего отобразить сайт
2. **docker.au-team.irpo** — должен отобразить testapp

Также проверьте:
```bash
curl http://web.au-team.irpo -u WEB:P@ssw0rd
curl http://docker.au-team.irpo
```

> [!IMPORTANT]
> Убедитесь, что DNS на HQ-CLI настроен на HQ-SRV (192.168.1.2) и все записи корректно резолвятся:
> ```bash
> nslookup web.au-team.irpo
> nslookup docker.au-team.irpo
> ```

wget https://browser.yandex.ru/download/?banerid=6302000000&zih=1&beta=1&os=linux&x64=1&package=deb&full=1
 sudo dpkg -i ~/Загрузки/yandex*.deb
 sudo apt install -f
 yandex-browser
