# nginx — обратный прокси на ISP

Сначала с ISP проверь оба маршрутизатора на порту 8080 (см. [проброс портов](./08-port-forwarding.md)).

```bash
apt-get install -y nginx
```

В `/etc/nginx/sites-available.d/default.conf`:

```nginx
server {
    listen 80;
    server_name web.au-team.irpo;
    location / {
        proxy_pass http://172.16.1.2:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80;
    server_name docker.au-team.irpo;
    location / {
        proxy_pass http://172.16.2.2:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Если ссылки еще нет:

```bash
ln -s /etc/nginx/sites-available.d/default.conf /etc/nginx/sites-enabled.d/default.conf
nginx -t
systemctl enable --now nginx
systemctl reload nginx
```

На HQ-CLI добавь в `/etc/hosts` (сохрани остальные записи):

```text
172.16.1.1 web.au-team.irpo
172.16.2.1 docker.au-team.irpo
```

Это нужно, если DNS Samba не содержит нужных записей. Проверь:

```bash
getent hosts web.au-team.irpo docker.au-team.irpo
curl -I http://web.au-team.irpo
curl -I http://docker.au-team.irpo
```

Открой оба адреса в браузере: web → сайт HQ-SRV, docker → testapp BR-SRV.

> **Важно:** имена должны вести на ISP, где слушает nginx, а upstream — на внешний адрес соответствующего маршрутизатора:8080. Удали противоречащие записи этих имен в hosts. При `502` проверь upstream с ISP; при чужой странице — server_name, разрешение имени и дубли конфигураций. Путь `sites-enabled.d` в исходнике был склеен неверно.
