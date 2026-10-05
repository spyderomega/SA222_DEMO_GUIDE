# Межсетевой экран — HQ-RTR и BR-RTR

Сначала добейся работающих IPsec и OSPF. Фильтрация должна применяться **на вход WAN `isp`**, а не только tunnel.0.

Из `configure terminal` на обоих маршрутизаторах убери разрешение всего трафика из IPsec-карты и сохрани разрешение внутреннего трафика туннеля отдельно:

```text
no filter-map ipv4 FMAP 30
filter-map ipv4 TUNNEL 10
 match any any any
 set accept
exit
interface tunnel.0
 set filter-map in TUNNEL 20
exit
```

Настрой WAN-карту. `<PEER_IP>` — WAN второго маршрутизатора, `<LOCAL_IP>` — свой WAN, `<ISP_IP>` — свой ISP-шлюз. Первые два значения — из [таблицы IPsec](./03-ipsec.md); ISP: HQ — `172.16.1.1`, BR — `172.16.2.1`.

```text
filter-map ipv4 WAN 10
 match udp host <PEER_IP> eq 500 host <LOCAL_IP> eq 500
 set accept
exit
filter-map ipv4 WAN 20
 match tcp any any ack
 set accept
exit
filter-map ipv4 WAN 30
 match udp host 77.88.8.7 eq 53 any range 1024 65535
 match udp host 77.88.8.3 eq 53 any range 1024 65535
 match udp host <ISP_IP> eq 123 any
 set accept
exit
filter-map ipv4 WAN 40
 match tcp host <ISP_IP> any eq 80
 match tcp host <ISP_IP> any eq 443
 match tcp host <ISP_IP> any eq 8080
 match tcp host <ISP_IP> any eq 2026
 set accept
exit
filter-map ipv4 WAN 50
 match icmp any any
 set accept
exit
interface isp
 set filter-map in FMAP 10
 set filter-map in WAN 20
exit
```

> **Важно:** FMAP обрабатывает IPsec до WAN-правил, TUNNEL пропускает внутренний трафик после этой обработки. В конце привязанных карт действует неявный запрет. UDP 4500 уже обрабатывается FMAP; если используется ESP без NAT-T, добавь соответствующую обработку протокола 50 для peer. Источники DNS/NTP замени фактическими: ответы от других серверов будут запрещены.

> **Ответный трафик:** `ack` разрешает TCP-ответы, но это проверка флага, а не отслеживание соединений. Не копируй из исходника разрешение *любых* входящих TCP/UDP на весь диапазон высоких портов: оно открывает лишние подключения. Для других нужных UDP-служб добавляй конкретные источники/порты. AD, NFS, печать, логи и метрики между офисами идут внутри туннеля.

С ISP проверь веб-пробросы и SSH; с клиентов — HTTP/HTTPS, DNS, NTP, доменный вход и связь HQ↔BR. Проверь запрет **нового** TCP-подключения на другой, заведомо работающий внутренний сервис. На маршрутизаторе:

```text
show filter-map ipv4
show counters interface isp filter-map in
show crypto-ipsec ike security-associations
show ip ospf neighbor
```

После успешной проверки: `write memory`. Если сервис сломался, проверь счетчики нужного правила и направление трафика, не возвращай общее `accept any`.

[Порядок filter-map и неявный запрет в EcoRouter](https://docs.ecorouter.ru/Руководство/21-Списки-доступа/03-Filter-map/02-Настройка-L3-filter-map).
