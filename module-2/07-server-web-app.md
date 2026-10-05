# Apache + MariaDB — HQ-SRV

## Файлы и службы

```bash
apt-get install -y lamp-server
lsblk
mount /dev/sr0 /mnt
ls /mnt/web
cp /mnt/web/index.php /var/www/html/
cp -r /mnt/web/images /var/www/html/
systemctl enable --now mariadb
mariadb -u root
```

Если ISO уже смонтирован, используй текущий путь. Если в твоем наборе есть `logo.png`, скопируй и его; пути картинок сверь с `index.php`.

## В консоли MariaDB

```sql
CREATE DATABASE webdb;
CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost';
EXIT;
```

> **Важно:** по заданию пользователь **web**, а не `webc` из старого примера. Если БД/пользователь уже существуют, проверь их вместо повторного создания. Не выдавай права на другие БД.

## Импорт и подключение приложения

```bash
file /mnt/web/dump.sql
mariadb -u web -p -D webdb < /mnt/web/dump.sql
```

Введи пароль `P@ssw0rd`. Только если `file` показывает UTF-16LE, перед импортом конвертируй дамп:

```bash
iconv -f UTF-16LE -t UTF-8 /mnt/web/dump.sql -o /tmp/dump_utf8.sql
mariadb -u web -p -D webdb < /tmp/dump_utf8.sql
```

Не импортируй обе версии подряд. В `/var/www/html/index.php` найди параметры подключения и установи:

| Параметр | Значение |
| --- | --- |
| Сервер БД | localhost |
| База | webdb |
| Пользователь | web |
| Пароль | P@ssw0rd |

Запуск и проверка:

```bash
mariadb -u web -p -D webdb -e "SHOW TABLES;"
systemctl enable --now httpd2
systemctl status httpd2
curl -I http://127.0.0.1
```

С HQ-CLI открой `http://192.168.100.2`: должны отображаться данные БД и изображения.

> **Если ошибка:** проверь реквизиты в PHP, импорт таблиц и журнал `journalctl -u httpd2 -n 30`. Если служба уже запущена и менял ее конфигурацию, примени изменения отдельно; `enable --now` не перезапускает работающую службу.

В отчет: Apache, MariaDB, имя БД/пользователя, импорт и проверка сайта.
