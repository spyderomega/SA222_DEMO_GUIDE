# Базовая настройка

## Имена

Альт: выполни на каждой машине со своим именем (см. [таблицу](./00-overview.md)).

```bash
hostnamectl set-hostname hq-srv.au-team.irpo
hostnamectl
```

EcoRouter: на HQ-RTR; на BR-RTR замени имя на `br-rtr`.

```text
enable
configure terminal
hostname hq-rtr
ip domain-name au-team.irpo
write memory
```

## IPv4 на Альт (HQ-SRV, BR-SRV)

Определи интерфейс по MAC-адресу:

```bash
ip -c a
```

Создай каталог `/etc/net/ifaces/<ИНТЕРФЕЙС>/`, если его нет. Файл `options`:

```text
TYPE=eth
BOOTPROTO=static
DISABLED=no
NM_CONTROLLED=no
SYSTEMD_CONTROLLED=no
```

В том же каталоге запиши следующие файлы (пример HQ-SRV):

| Файл | Содержимое |
| --- | --- |
| `ipv4address` | `192.168.100.2/27` |
| `ipv4route` | `default via 192.168.100.1` |
| `resolv.conf` | две строки: `search au-team.irpo` и `nameserver 192.168.100.2` |

На BR-SRV: адрес `192.168.0.2/28`, шлюз `192.168.0.1`, DNS тот же.

```bash
systemctl restart network
ip -c a
ip -c r
cat /etc/resolv.conf
```

> **Важно:** перезапуск сети может оборвать SSH — меняй сеть через консоль. Шлюз HQ-SRV проверяй после настройки VLAN, DNS — после раздела DNS. Для установки пакетов до запуска HQ-SRV DNS временно используй доступный внешний DNS.

## ISP: адресация

В примере `ens19` → Интернет, `ens20` → HQ, `ens21` → BR. Сначала сверь MAC-адреса.

В `/etc/net/ifaces/ens19/options`: `TYPE=eth`, `BOOTPROTO=dhcp`. Для `ens20` и `ens21` — `options` как выше со статической настройкой.

```bash
mkdir -p /etc/net/ifaces/ens20 /etc/net/ifaces/ens21
echo "172.16.1.1/28" > /etc/net/ifaces/ens20/ipv4address
echo "172.16.2.1/28" > /etc/net/ifaces/ens21/ipv4address
systemctl restart network
ip -c a
ip -c r
ping -c 3 77.88.8.7
```

> **Важно:** адрес BR записывается в `ens21`, а не повторно в `ens20` — в исходнике опечатка. Маршрут по умолчанию ISP должен вести к провайдеру, обычно он приходит по DHCP.

## ISP: forwarding и NAT

В `/etc/net/sysctl.conf` установи `net.ipv4.ip_forward = 1`.

```bash
systemctl restart network
apt-get update
apt-get install -y iptables
iptables -t nat -A POSTROUTING -s 172.16.1.0/28 -o ens19 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 172.16.2.0/28 -o ens19 -j MASQUERADE
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
sysctl net.ipv4.ip_forward
iptables -t nat -L -n -v
```

> **Важно:** `-A` добавляет правило — не повторяй без проверки. Сохраняй правила через `>`, чтобы не дописывать несколько копий дампа. Если forwarding включен, но трафик не идет, проверь цепочку FORWARD.

## EcoRouter: WAN

HQ-RTR, порт `ge0` → ISP:

```text
interface isp
description "ISP"
ip address 172.16.1.2/28
exit
ip route 0.0.0.0/0 172.16.1.1
port ge0
service-instance ge0/isp
encapsulation untagged
connect ip interface isp
exit
exit
write memory
```

На BR-RTR выполни тот же блок, заменив адрес на `172.16.2.2/28`, шлюз — на `172.16.2.1`.

## EcoRouter: локальные сети

### HQ-RTR

Три VLAN через один порт `ge1`.

```text
interface vl100
description "VLAN 100"
ip address 192.168.100.1/27
exit
interface vl200
description "VLAN 200"
ip address 192.168.200.1/24
exit
interface vl999
description "VLAN 999"
ip address 192.168.99.1/29
exit
port ge1
service-instance ge1/vl100
encapsulation dot1q 100 exact
rewrite pop 1
connect ip interface vl100
exit
service-instance ge1/vl200
encapsulation dot1q 200 exact
rewrite pop 1
connect ip interface vl200
exit
service-instance ge1/vl999
encapsulation dot1q 999 exact
rewrite pop 1
connect ip interface vl999
exit
exit
write memory
```

### BR-RTR

```text
port ge1
service-instance ge1/int1
encapsulation untagged
exit
exit
interface int1
description "BR-Net"
ip address 192.168.0.1/28
connect port ge1 service-instance ge1/int1
exit
write memory
```

> **Важно:** блок HQ выполняй только на HQ-RTR, блок BR (с `int1`) — только на BR-RTR. Физический порт связан с L3-интерфейсом через `service-instance` и `connect`; без этой связи интерфейс не поднимется. В примере BR исправлена несогласованность `te1`/`ge1`.

Проверка из привилегированного режима:

```text
show port brief
show service-instance brief
show ip interface brief
show ip route
```

Проверь ping до своего ISP-шлюза, затем до `77.88.8.7`.
