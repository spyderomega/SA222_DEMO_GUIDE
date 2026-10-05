# DNS — HQ-SRV (BIND)

```bash
apt-get update
apt-get install -y bind bind-utils
```

В `/var/lib/bind/etc/options.conf` настрой `listen-on`, `forwarders`, `allow-query` по примеру. Сохрани остальные необходимые параметры файла.

![options.conf](../assets/page-071-img-01.jpeg)

Добавь определения прямой и обратных зон в конец `/var/lib/bind/etc/rfc1912.conf`, сохранив существующие зоны:

![Определения зон](../assets/page-071-img-02.jpeg)

```bash
cp /var/lib/bind/etc/zone/empty /var/lib/bind/etc/zone/au-team.irpo
cp /var/lib/bind/etc/zone/empty /var/lib/bind/etc/zone/100.168.192.in-addr.arpa
cp /var/lib/bind/etc/zone/empty /var/lib/bind/etc/zone/200.168.192.in-addr.arpa
```

Заполни файлы зон по образцам:

![Прямая зона au-team.irpo](../assets/page-072-img-01.jpeg)
![Обратная зона 192.168.100](../assets/page-072-img-02.jpeg)
![Обратная зона 192.168.200](../assets/page-073-img-01.jpeg)

Обязательные записи:

| Имя в au-team.irpo | A | PTR |
| --- | --- | --- |
| hq-rtr | 192.168.100.1 | да |
| br-rtr | 192.168.0.1 | — |
| hq-srv | 192.168.100.2 | да |
| hq-cli | фактический IP HQ-CLI | да |
| br-srv | 192.168.0.2 | — |
| docker | 172.16.1.1 | — |
| web | 172.16.2.1 | — |

Ключ rndc: в исходном практикуме используется следующий способ (не перезаписывай уже настроенный ключ):

```bash
rndc-confgen > /etc/bind/rndc.key
sed -i '6,$d' /etc/bind/rndc.key
```

> **Важно:** после обрезки проверь, что `rndc.key` содержит полный блок `key { ... };`. Типографские кавычки из PDF заменены обычными.

Права, проверка и запуск:

```bash
chown root:named /var/lib/bind/etc/zone/au-team.irpo /var/lib/bind/etc/zone/100.168.192.in-addr.arpa /var/lib/bind/etc/zone/200.168.192.in-addr.arpa
named-checkconf
named-checkconf -z
systemctl enable --now bind.service
systemctl status bind
```

> **Важно:** если BIND уже запущен, после успешной проверки примени изменения через `systemctl restart bind`. `enable --now` не перечитывает конфигурацию уже работающей службы. При ошибках проверь `journalctl -u bind -n 30`.

Проверка с клиента:

```bash
host hq-srv.au-team.irpo 192.168.100.2
host 192.168.100.2 192.168.100.2
host hq-rtr.au-team.irpo 192.168.100.2
host 192.168.100.1 192.168.100.2
host hq-cli.au-team.irpo 192.168.100.2
host <IP_HQ-CLI> 192.168.100.2
host br-rtr.au-team.irpo 192.168.100.2
host br-srv.au-team.irpo 192.168.100.2
host docker.au-team.irpo 192.168.100.2
host web.au-team.irpo 192.168.100.2
host ya.ru 192.168.100.2
```

> **Важно:** FQDN в SOA/NS/PTR заканчивается точкой; увеличивай serial после изменения зоны. A и PTR HQ-CLI должны совпадать с его DHCP-адресом. Разреши запросы обеих офисных сетей, проверь UDP/TCP 53 и DNS `192.168.100.2` на клиентах.
