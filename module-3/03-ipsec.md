# GRE over IPsec — HQ-RTR и BR-RTR

Сначала проверь GRE `tunnel.0` и соседство OSPF. Если исходный туннель IP in IP, сначала переведи **обе** стороны в GRE, сохранив адреса туннеля. Команда `ip tunnel`: HQ — `172.16.1.2 172.16.2.2 mode gre`, BR — наоборот.

Из `configure terminal` выполни общий блок на каждом маршрутизаторе, заменив параметры:

| Узел | LOCAL_IP | PEER_IP |
| --- | --- | --- |
| HQ-RTR | 172.16.1.2 | 172.16.2.2 |
| BR-RTR | 172.16.2.2 | 172.16.1.2 |

```text
crypto-ipsec ike enable
crypto-ipsec profile IPSEC ike-v2
 mode tunnel
 nat-traversal
 ike-phase1
  proposal aes256-sha256-modp2048
  auth pre-shared-key P@ssw0rd
 exit
 ike-phase2
  protocol esp
  proposal aes256-sha256
  local-ts <LOCAL_IP>
  remote-ts <PEER_IP>
 exit
exit
crypto-map CMAP 10
 match peer <PEER_IP>
 set crypto-ipsec profile IPSEC
exit
filter-map ipv4 FMAP 10
 match gre host <LOCAL_IP> host <PEER_IP>
 set crypto-map CMAP peer <PEER_IP>
exit
filter-map ipv4 FMAP 20
 match udp host <PEER_IP> eq 4500 host <LOCAL_IP> eq 4500
 set crypto-map CMAP peer <PEER_IP>
exit
filter-map ipv4 FMAP 30
 match any any any
 set accept
exit
interface isp
 set filter-map in FMAP 10
exit
interface tunnel.0
 ip mtu 1360
 set filter-map in FMAP 10
exit
write memory
```

> **Важно:** на BR свои local-ts, peer и направления match — в исходнике скопированы значения HQ. Имя `isp` сверь с WAN-интерфейсом. Оба конца используют одинаковые алгоритмы/PSK и MTU 1360. Вложения `ike-phase1/2` и `exit` сверь с контекстом CLI своей версии; каждая команда начинается в указанном режиме.

Проверка:

```text
show crypto-ipsec ike connections
show crypto-ipsec ike security-associations
show ip ospf neighbor
show ip route ospf
show counters interface isp filter-map in
```

IKE — `ESTABLISHED`, дочерняя SA — `INSTALLED`, OSPF — FULL. Ping до второго адреса туннеля и сервера другого офиса должен работать. Убедись по SA/перехвату WAN, что трафик зашифрован, а не только проходит по GRE.

При сохранении адресов и имени tunnel.0 объявления OSPF остаются прежними; при их изменении исправь `network`, `no passive-interface` и аутентификацию. Не включай соседство на WAN.

В отчет: GRE over IPsec, причина выбора, IKEv2/ESP, алгоритмы, MTU, адреса и изменения OSPF. [Пример EcoRouter](https://docs.ecorouter.ru/Руководство/24-IPsec/04-Настройка-GRE-over-IPsec).
