# SSH — HQ-SRV и BR-SRV

В `/etc/openssh/sshd_config` установи:

```text
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/openssh/banner
```

```bash
echo "Authorized access only" > /etc/openssh/banner
sshd -t
systemctl restart sshd
systemctl status sshd
ss -lntp | grep :2026
```

> **Важно:** проверь синтаксис через `sshd -t` до перезапуска. Изменяй существующие параметры, проверь дубли и секции `Match`. Текущую сессию оставь открытой до успешного нового входа; TCP 2026 должен быть доступен через firewall.

С другой машины:

```bash
ssh -p 2026 sshuser@192.168.100.2
ssh -p 2026 sshuser@192.168.0.2
```

Проверь баннер; вход другого пользователя и подключение на порт 22 должны быть запрещены. При двух неверных паролях соединение должно закрыться (`MaxAuthTries` действует в рамках одного соединения).
