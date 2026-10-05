# Защита SSH — HQ-SRV

```bash
apt-get install -y fail2ban iptables rsyslog
```

В `/etc/openssh/sshd_config` установи (сохрани Port 2026 и AllowUsers):

```text
SyslogFacility AUTHPRIV
LogLevel INFO
```

В rsyslog добавь один раз локальное правило `authpriv.* /var/log/auth.log`; оно не пересылает логи самому себе.

```bash
rsyslogd -N1
systemctl enable --now rsyslog
systemctl restart rsyslog
sshd -t
systemctl restart sshd
```

Создай `/etc/fail2ban/jail.d/sshd.local`:

```ini
[sshd]
enabled = true
port = 2026
backend = auto
logpath = /var/log/auth.log
banaction = iptables-multiport
maxretry = 3
findtime = 10m
bantime = 1m
```

```bash
fail2ban-client -t
systemctl enable --now fail2ban
systemctl restart fail2ban
fail2ban-client status sshd
```

С тестового клиента сделай **три неверные попытки входа** на 2026; при MaxAuthTries 2 понадобятся несколько SSH-соединений. На HQ-SRV проверь появление ошибок в `/var/log/auth.log` и IP в `fail2ban-client status sshd`. Через минуту IP должен выйти из бана.

Для снятия бана только тестового адреса:

```bash
fail2ban-client set sshd unbanip 192.168.0.2
```

> **Важно:** тестируй с отдельного клиента, оставив консоль HQ-SRV открытой. Порт и logpath должны совпадать с фактическими. Если используется только journald, выбери `backend = systemd` и убери logpath. Не редактируй пакетный jail.conf — локальные настройки сохранятся после обновления.
