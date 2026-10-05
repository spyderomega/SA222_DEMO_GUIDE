# Логи warning и выше — HQ-SRV

Клиенты: HQ-RTR, BR-RTR, BR-SRV. HQ-SRV не пересылает логи самому себе.

## HQ-SRV: прием UDP 514

```bash
apt-get install -y rsyslog logrotate
mkdir -p /opt/hq-rtr /opt/br-rtr /opt/br-srv
```

Добавь в `/etc/rsyslog.conf` блок (или в подключаемый им каталог конфигураций). Модуль imudp загружай один раз:

```rsyslog
module(load="imudp")
input(type="imudp" port="514")
if ($fromhost-ip == "192.168.100.1" and $syslogseverity <= 4) then {
    action(type="omfile" file="/opt/hq-rtr/router.log")
    stop
}
if ($fromhost-ip == "10.10.10.2" and $syslogseverity <= 4) then {
    action(type="omfile" file="/opt/br-rtr/router.log")
    stop
}
if ($fromhost-ip == "192.168.0.2" and $syslogseverity <= 4) then {
    action(type="omfile" file="/opt/br-srv/server.log")
    stop
}
```

```bash
rsyslogd -N1
systemctl enable --now rsyslog
systemctl restart rsyslog
```

> **Важно:** IP в условиях — фактический источник syslog, сверь его по `tcpdump -ni any udp port 514`. Маршрутизатор может выбрать другой адрес. Используй точное равенство, не `contains`; оставь каталоги `hq-rtr`, `br-rtr`, `br-srv`. Если rsyslog работает не от root, дай его пользователю права записи в эти каталоги.

## HQ-RTR и BR-RTR

Из `configure terminal`:

```text
rsyslog host 192.168.100.2
write memory
```

Уровень warning и выше отбирается сервером.

## BR-SRV

Установи rsyslog. В конфигурацию добавь, сохранив существующие локальные модули/правила:

```rsyslog
*.warning @192.168.100.2:514
```

```bash
rsyslogd -N1
systemctl enable --now rsyslog
systemctl restart rsyslog
logger -p user.warning "check-br-srv"
```

Одна `@` — UDP, согласованный с приемником; `@@` означает TCP. На HQ-SRV проверь `tail /opt/br-srv/server.log`. Аналогично проверь поступление warning-событий от обоих маршрутизаторов.

## Ротация на HQ-SRV

В `/etc/logrotate.d/remote-logs`:

```text
/opt/*.log /opt/*/*.log {
    weekly
    minsize 10M
    compress
    rotate 4
    missingok
    notifempty
    sharedscripts
    postrotate
        /bin/systemctl kill -s HUP rsyslog.service
    endscript
}
```

```bash
logrotate -d /etc/logrotate.conf
systemctl list-timers --all | grep logrotate
ls /etc/cron.daily/
```

Должен работать штатный ежедневный запуск logrotate через timer **или** cron; если установлен `logrotate.timer`, включи `systemctl enable --now logrotate.timer`. Не включай несуществующую службу logrotate.

> **Важно:** weekly + minsize означает ротацию раз в неделю при размере не меньше 10 МБ. Маски покрывают /opt и созданные подкаталоги; при более глубоком размещении логов добавь соответствующие пути. HUP заставляет rsyslog открыть новый файл после ротации.
