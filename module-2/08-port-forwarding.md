# Проброс портов — HQ-RTR и BR-RTR

Сначала проверь работу приложений напрямую. На EcoRouter из `configure terminal`:

## HQ-RTR

```text
ip nat source static tcp 192.168.100.2 80 172.16.1.2 8080
ip nat source static tcp 192.168.100.2 2026 172.16.1.2 2026
write memory
```

## BR-RTR

```text
ip nat source static tcp 192.168.0.2 8080 172.16.2.2 8080
ip nat source static tcp 192.168.0.2 2026 172.16.2.2 2026
write memory
```

Проверка с ISP:

```bash
curl -I http://172.16.1.2:8080
curl -I http://172.16.2.2:8080
ssh -p 2026 sshuser@172.16.1.2
ssh -p 2026 sshuser@172.16.2.2
```

На маршрутизаторах: `show ip nat translations`.

> **Важно:** HQ: внешний 8080 → Apache 80; BR: внешний 8080 → опубликованный Docker 8080. В команде сначала внутренние IP/порт, затем внешние. NAT inside/outside и маршруты уже должны работать. Проверяй с ISP: обращение из LAN к своему внешнему адресу может требовать hairpin NAT.
