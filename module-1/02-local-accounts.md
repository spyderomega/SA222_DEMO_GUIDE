# Локальные пользователи

## HQ-SRV и BR-SRV (Альт)

```bash
useradd -m -u 2026 sshuser
passwd sshuser
usermod -aG wheel sshuser
visudo
```

Пароль: `P@ssw0rd`. В `visudo` добавь:

```text
sshuser ALL=(ALL:ALL) NOPASSWD: ALL
```

Проверка:

```bash
visudo -c
su - sshuser
id
sudo -n whoami
exit
```

Ожидается UID `2026` и ответ `root` без пароля.

> **Важно:** перед созданием проверь `id sshuser` и `getent passwd 2026`: пользователь или UID могут уже существовать. Для sudo используй `visudo`, чтобы проверить синтаксис; после добавления в wheel войди заново.

## HQ-RTR и BR-RTR (EcoRouter)

В режиме `configure terminal`:

```text
username net_admin
password P@ssw0rd
role admin
exit
write memory
```

Проверка: войди под `net_admin`, затем `show users localdb` — роль `admin`.

Если маршрутизатор работает на Linux: создай `net_admin`, задай `P@ssw0rd`, добавь в wheel и через `visudo` строку `net_admin ALL=(ALL:ALL) NOPASSWD: ALL`. UID 2026 требуется только для `sshuser` на серверах.
