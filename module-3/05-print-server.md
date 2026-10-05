# PDF-принтер — HQ-SRV → HQ-CLI

## HQ-SRV

```bash
apt-get update
apt-get install -y cups cups-pdf
```

В `/etc/cups/cupsd.conf` разреши сетевое прослушивание: вместо только `Listen localhost:631` — `Port 631`; сохрани строку Unix-сокета. Для доступа HQ настрой существующие секции (не дублируй их):

```text
<Location />
  Order allow,deny
  Allow localhost
  Allow 192.168.100.0/27
  Allow 192.168.200.0/24
</Location>
<Location /admin>
  AuthType Default
  Require user @SYSTEM
  Order allow,deny
  Allow localhost
  Allow 192.168.100.0/27
  Allow 192.168.200.0/24
</Location>
<Location /admin/conf>
  AuthType Default
  Require user @SYSTEM
  Order allow,deny
  Allow localhost
  Allow 192.168.100.0/27
  Allow 192.168.200.0/24
</Location>
```

```bash
cupsd -t
systemctl enable --now cups
systemctl restart cups
```

С HQ-CLI открой `https://hq-srv.au-team.irpo:631/admin/`, войди администратором HQ-SRV. Проверь очередь **Cups-PDF**; если отсутствует — добавь виртуальный CUPS-PDF-принтер. Включи общий доступ к очереди.

## HQ-CLI

Через настройки принтеров добавь сетевой принтер `ipp://hq-srv.au-team.irpo:631/printers/Cups-PDF`, выбери его по умолчанию. Имя очереди сверь в CUPS — регистр имеет значение.

Проверка: отправь тестовую страницу и найди PDF на HQ-SRV; путь вывода указан параметром `Out` в `/etc/cups/cups-pdf.conf`.

> **Если принтер недоступен:** проверь имя HQ-SRV, TCP 631, общий доступ к очереди и службу cups. Исключение для самоподписанного сертификата административного интерфейса CUPS не относится к требованию доверенных сертификатов web/docker.
