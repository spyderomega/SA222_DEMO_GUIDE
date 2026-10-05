# testapp в Docker — BR-SRV

## Образы с Additional.iso

```bash
apt-get install -y docker-engine docker-compose-v2
systemctl enable --now docker
lsblk
mount /dev/sr0 /mnt
ls /mnt/docker
cat /mnt/docker/readme.txt
docker load < /mnt/docker/site_latest.tar
docker load < /mnt/docker/mariadb_latest.tar
docker images
mkdir -p /opt/testapp
cd /opt/testapp
```

> **Важно:** если ISO уже смонтирован, используй его текущий путь. Названия файлов и теги образов сверь с ISO и `docker images`: не рассчитывай на скачивание образов из Интернета.

## `/opt/testapp/compose.yaml`

Пример для образов из практикума, переменные которых указаны в `readme.txt`:

```yaml
services:
  database:
    image: mariadb:latest
    container_name: db
    restart: always
    environment:
      DB_NAME: appdb_maria
      DB_USER: webapp
      DB_PASS: "P@ss2026-db"
      MARIADB_ROOT_PASSWORD: "P@ssw0rd"
    volumes:
      - db_data:/var/lib/mysql

  app:
    image: site:latest
    container_name: testapp
    restart: always
    ports:
      - "8080:8000"
    environment:
      DB_TYPE: maria
      DB_HOST: database
      DB_PORT: "3306"
      DB_NAME: appdb_maria
      DB_USER: webapp
      DB_PASS: "P@ss2026-db"
    depends_on:
      - database

volumes:
  db_data:
```

> **Не перепутай:** по заданию БД — `appdb_maria`, пользователь `webapp`, пароль `P@ss2026-db`. В старом примере другие значения. `DB_TYPE: maria` — значение переменной образа для MariaDB, сверь с readme. Эти ISO-образы используют `DB_*`; у стандартного образа MariaDB переменные другие.

> **Порты:** в readme практикума приложение слушает **8000**, снаружи нужен **8080**, поэтому `8080:8000`. Если твой образ слушает другой порт, измени правую часть. `DB_HOST: database` — имя сервиса внутри Compose; 3306 — внутренний порт БД. Проверь теги image и имя контейнера по своему заданию (в исходнике встречается опечатка `tespapp`).

## Запуск и проверка

```bash
docker compose config
docker compose up -d
docker compose ps
docker compose logs --tail=50
curl -I http://127.0.0.1:8080
docker compose restart
```

С HQ-CLI открой `http://192.168.0.2:8080`: проверь страницу и работу с данными. Затем проверь после перезагрузки BR-SRV.

> **Если БД недоступна:** `depends_on` задает порядок запуска, но не ожидает готовности БД; проверь логи и повторный запуск app после готовности database. Volume сохраняет БД, не удаляй его через `docker compose down -v`. Изменение переменных не обязательно меняет уже созданную БД — проверь реальные учетные данные.
