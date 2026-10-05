# DHCP — HQ-RTR → HQ-CLI

В режиме `configure terminal`:

```text
ip pool VLAN200 192.168.200.2-192.168.200.254
dhcp-server 1
pool VLAN200 1
mask 24
gateway 192.168.200.1
dns 192.168.100.2
domain-name au-team.irpo
exit
exit
interface vl200
dhcp-server 1
exit
write memory
```

Проверка на HQ-RTR:

```text
show running-config dhcp-server 1
show dhcp-server clients vl200
```

На HQ-CLI включи автоматическое получение IPv4/DNS в используемом сетевом менеджере. Для etcnet — `BOOTPROTO=dhcp` в `/etc/net/ifaces/<ИНТЕРФЕЙС>/options`, затем `systemctl restart network`.

```bash
ip -c a
ip -c r
cat /etc/resolv.conf
```

Ожидается адрес `192.168.200.2–254/24`, шлюз `192.168.200.1`, DNS `192.168.100.2`, суффикс `au-team.irpo`.

> **Важно:** `.1` исключен из пула — это шлюз. Сервер привязан к `vl200`; проверь VLAN 200 и отсутствие старой статической настройки клиента. DNS заработает после настройки HQ-SRV. Для A/PTR HQ-CLI используй фактически выданный адрес и обеспечь его постоянство, если запись статическая.

Настройки DHCP внеси в отчет.
