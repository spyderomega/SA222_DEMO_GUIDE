# OSPF — HQ-RTR и BR-RTR

Настрой после проверки туннеля, в режиме `configure terminal`.

## HQ-RTR

```text
router ospf 1
ospf router-id 10.10.10.1
passive-interface default
no passive-interface tunnel.0
network 10.10.10.0/30 area 0
network 192.168.100.0/27 area 0
network 192.168.200.0/24 area 0
network 192.168.99.0/29 area 0
exit
interface tunnel.0
ip ospf authentication message-digest
ip ospf message-digest-key 1 md5 P@ssw0rd
exit
write memory
```

## BR-RTR

```text
router ospf 1
ospf router-id 10.10.10.2
passive-interface default
no passive-interface tunnel.0
network 192.168.0.0/28 area 0
network 10.10.10.0/30 area 0
exit
interface tunnel.0
ip ospf authentication message-digest
ip ospf message-digest-key 1 md5 P@ssw0rd
exit
write memory
```

Проверка из привилегированного режима:

```text
show ip ospf neighbor
show ip route ospf
show ip ospf interface tunnel.0
show ip ospf interface
```

Ожидается соседство FULL и маршруты к сетям второго офиса. С HQ-SRV проверь `ping 192.168.0.2`, с BR-SRV — `ping 192.168.100.2`.

> **Важно:** только `tunnel.0` активен для соседства, LAN объявляются как passive. На обоих концах должны совпадать area, тип аутентификации, номер ключа и пароль; router-id должны отличаться. Если соседство не поднимается, сначала проверь туннель и MTU.

Конфигурацию и парольную защиту внеси в отчет.
