# HTTP-аутентификация WEB — ISP

Настрой после работающего nginx-прокси.

```bash
apt-get install -y apache2
htpasswd -c /etc/nginx/.htpasswd WEB
```

Введи пароль `P@ssw0rd` дважды. Apache здесь нужен для утилиты `htpasswd`; службу httpd2 на ISP не запускай, порт 80 использует nginx.

В существующий `location /` **только** сервера `web.au-team.irpo` добавь, сохранив proxy_pass и заголовки:

```nginx
auth_basic "Restricted access";
auth_basic_user_file /etc/nginx/.htpasswd;
```

```bash
nginx -t
systemctl reload nginx
```

С HQ-CLI:

```bash
curl -I http://web.au-team.irpo
curl -u WEB -I http://web.au-team.irpo
```

Первый запрос — `401` с запросом аутентификации; второй после ввода пароля — ответ приложения. В браузере проверь отказ с неверным паролем и успешный вход `WEB / P@ssw0rd`. `docker.au-team.irpo` должен открываться как раньше.

> **Важно:** `-c` создает/перезаписывает файл. Для изменения пароля существующего WEB используй `htpasswd /etc/nginx/.htpasswd WEB` без `-c`. Пользователь nginx должен читать файл; при `403/500` проверь права и error log. Имя WEB чувствительно к регистру.
