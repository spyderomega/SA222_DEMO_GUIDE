# Samba DC — BR-SRV; клиент — HQ-CLI

## 1. Контроллер домена на BR-SRV

Проверь FQDN `br-srv.au-team.irpo` и синхронизацию времени.

```bash
apt-get update
apt-get install -y task-samba-dc bind-utils
for service in smb nmb krb5kdc slapd bind; do
    systemctl disable --now "$service"
done
```

> **Важно:** останавливай конфликтующие службы только на BR-SRV. Если служба отсутствует, это не ошибка настройки домена. Старый домен и базы Samba автоматически не удаляй; очистка из исходника уничтожает существующий домен.

На чистом стенде:

```bash
samba-tool domain provision
```

Ответы при развертывании:

| Поле | Значение |
| --- | --- |
| Realm | AU-TEAM.IRPO |
| Domain | AU-TEAM |
| Server Role | dc |
| DNS backend | SAMBA_INTERNAL |
| DNS forwarder | доступный внешний DNS, например 77.88.8.7 |
| Administrator password | свой сложный пароль, запомни для ввода HQ-CLI |

Не принимай автоматически неверные значения. После успешного создания:

```bash
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl enable --now samba
systemctl status samba
```

Установи для BR-SRV DNS `127.0.0.1` и search `au-team.irpo` в настройках сети, чтобы они сохранялись после перезапуска. Проверка:

```bash
samba-tool domain info 127.0.0.1
host -t SRV _ldap._tcp.au-team.irpo 127.0.0.1
host -t SRV _kerberos._udp.au-team.irpo 127.0.0.1
kinit Administrator@AU-TEAM.IRPO
klist
```

> **Если Kerberos не работает:** проверь DNS, время, регистр realm и пароль Administrator. Пароль должен соответствовать политике сложности.

## 2. Группа и пять пользователей на BR-SRV

```bash
samba-tool group add hq
for i in 1 2 3 4 5; do
    samba-tool user add "hquser$i" 'P@ssw0rd'
    samba-tool user setexpiry "hquser$i" --noexpiry
    samba-tool group addmembers hq "hquser$i"
done
samba-tool group listmembers hq
```

Ожидаются `hquser1`–`hquser5`. Если объекты уже существуют, проверь их, не запускай создание повторно.

## 3. Ввод HQ-CLI в домен

На HQ-CLI сохрани корректные IPv4/шлюз, DNS установи **192.168.0.2**. Если закрепляешь статический адрес из примера, исключи конфликт с DHCP.

```bash
host br-srv.au-team.irpo 192.168.0.2
host -t SRV _ldap._tcp.au-team.irpo 192.168.0.2
apt-get update
apt-get install -y task-auth-ad-sssd
```

В ЦУС → **Пользователи → Аутентификация**: Домен Active Directory, домен `au-team.irpo`, рабочая группа `AU-TEAM`, имя компьютера `hq-cli`, **SSSD (в единственном домене)**. Применить → ввести `Administrator` и пароль домена → после успешного ввода перезагрузить HQ-CLI.

## 4. Доступ группы hq и только cat, grep, id

На HQ-CLI, от root:

```bash
apt-get install -y libnss-role
control libnss-role
```

Если модуль выключен, включи его через `control libnss-role enabled`. Затем:

```bash
roleadd hq wheel
visudo
```

В `/etc/sudoers` оставь определение `WHEEL_USERS`, задай разрешенные команды и ограниченное правило для wheel:

```sudoers
User_Alias WHEEL_USERS = %wheel
Cmnd_Alias SHELLCMD = /bin/cat, /bin/grep, /usr/bin/id
WHEEL_USERS ALL=(ALL:ALL) SHELLCMD
```

> **Важно:** не дублируй уже существующий alias. Замени широкое правило `WHEEL_USERS ... ALL` ограниченным; правило `NOPASSWD: ALL` для wheel должно быть выключено. Проверь также другие группы и индивидуальные правила: разрешения sudo суммируются. Пути cat/grep/id сверь через `command -v cat grep id`.

```bash
visudo -c
```

Выйди из локальной учетной записи, войди на HQ-CLI как `hquser1` (при необходимости `hquser1@au-team.irpo`). В терминале:

```bash
id
sudo -l
sudo cat /etc/hostname
sudo grep root /etc/passwd
sudo id
sudo ls /root
```

Первые три команды sudo разрешены, `sudo ls` запрещена. Проверь вход остальных пользователей hq. Если вход не работает, проверь `systemctl status sssd`, DNS и время.
