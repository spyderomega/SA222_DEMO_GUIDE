# Время — chrony на ISP

## ISP

```bash
apt-get install -y chrony
```

В `/etc/chrony.conf` замени активные строки источников этим блоком (остальные служебные параметры сохрани):

```text
server ntp5.ntp-servers.net iburst prefer minstratum 4
local stratum 5
allow 172.16.1.0/28
allow 172.16.2.0/28
allow 192.168.100.0/27
allow 192.168.200.0/24
allow 192.168.0.0/28
```

```bash
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources -v
chronyc tracking
```

> **Стратум:** `minstratum 4` задает нижнюю границу страты источника, ISP будет минимум stratum 5; `local stratum 5` — резервный локальный источник. Проверь фактическое `Stratum` через `chronyc tracking`: если внешний источник выше 4, ISP тоже будет выше 5. Для требования ровно 5 нужен выбранный источник со стратумом не выше 4. [Описание chrony](https://chrony-project.org/doc/latest/chrony.conf.html).

## HQ-SRV, HQ-CLI, BR-SRV (Альт)

Установи `chrony`. В `/etc/chrony.conf` оставь единственный источник: на HQ — `server 172.16.1.1 iburst`, на BR — `server 172.16.2.1 iburst`.

```bash
systemctl enable --now chronyd
systemctl restart chronyd
chronyc sources -v
chronyc tracking
```

Ожидается выбранный ISP-источник (`^*`); синхронизация занимает некоторое время.

## BR-RTR (EcoRouter)

Из `configure terminal`:

```text
ntp server 172.16.2.1
write memory
```

Проверка из привилегированного режима: `show ntp status`.

> **Важно:** NTP-адрес — ISP, не LAN-шлюз клиента. При отсутствии синхронизации проверь UDP 123, маршруты и `allow`; адреса в allow должны соответствовать источнику пакетов после NAT. Проверь время до Kerberos и ввода HQ-CLI в домен.
