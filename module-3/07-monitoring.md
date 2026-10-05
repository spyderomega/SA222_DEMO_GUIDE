# Мониторинг — HQ-SRV

Prometheus собирает метрики HQ-SRV/BR-SRV, Grafana показывает ЦП, занятую ОП и основной диск. Доступ HQ-CLI — `http://mon.au-team.irpo`.

## Метрики и сборщик

HQ-SRV:

```bash
apt-get install -y grafana prometheus prometheus-node_exporter nginx
systemctl enable --now prometheus-node_exporter
```

BR-SRV:

```bash
apt-get install -y prometheus-node_exporter
systemctl enable --now prometheus-node_exporter
```

В `/etc/prometheus/prometheus.yml` добавь отдельный job в существующий `scrape_configs`:

```yaml
scrape_configs:
  - job_name: servers
    static_configs:
      - targets: ["192.168.100.2:9100", "192.168.0.2:9100"]
```

```bash
promtool check config /etc/prometheus/prometheus.yml
systemctl enable --now prometheus grafana-server
systemctl restart prometheus
curl -I http://192.168.100.2:9100/metrics
curl -I http://192.168.0.2:9100/metrics
```

> **Важно:** не создавай второй ключ `scrape_configs`, вставь job в имеющийся список. В исходнике `hr-srv` — опечатка. Проверь оба targets в Prometheus (`http://192.168.100.2:9090/targets`): состояние UP.

В Grafana (`http://192.168.100.2:3000`) войди `admin / admin`, задай `P@ssw0rd`. Добавь источник **Prometheus**, URL `http://127.0.0.1:9090`, Save & test. Импортируй совместимый Node Exporter dashboard (в практикуме №11074; без Интернета — его JSON). Проверь метрики обоих серверов и основной filesystem, а не только tmpfs.

## URL mon и доступ только HQ

Добавь A-запись `mon.au-team.irpo → 192.168.100.2` в DNS, которым пользуется HQ-CLI (после ввода в домен это Samba на BR-SRV). В Samba:

```bash
samba-tool dns add 127.0.0.1 au-team.irpo mon A 192.168.100.2 -U Administrator
```

Эта команда выполняется **на BR-SRV**. Если запись уже существует, проверь ее вместо повторного добавления.

На HQ-SRV порт 80 уже занят Apache. Найди Listen/VirtualHost (`rg -n 'Listen|VirtualHost' /etc/httpd2`) и переведи Apache на `127.0.0.1:8081`, сохранив сайт. Проверь его локально. nginx будет слушать 80 и обслуживать два имени:

```nginx
server {
    listen 80 default_server;
    server_name hq-srv.au-team.irpo web.au-team.irpo;
    location / { proxy_pass http://127.0.0.1:8081; }
}
server {
    listen 80;
    server_name mon.au-team.irpo;
    allow 192.168.100.0/27;
    allow 192.168.200.0/24;
    deny all;
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
    }
}
```

Размести блоки в sites-available.d и подключи ссылкой в sites-enabled.d, как в [модуле 2](../module-2/09-reverse-proxy.md). Удали конфликтующий default_server. Проверь конфигурацию Apache, перезапусти httpd2, затем `nginx -t` и включи/перезапусти nginx.

> **Важно:** в `/etc/grafana/grafana.ini` задай `http_addr = 127.0.0.1` в `[server]`, перезапусти grafana-server: иначе порт 3000 обходит ограничения nginx. Разреши 9090 только для администрирования HQ, 9100 — сборщику HQ-SRV через firewall серверов. Не создавай внешние пробросы этих портов или имени mon. Существующий NAT HQ:8080 → HQ-SRV:80 продолжает обслуживать веб через первый server-блок.

Проверь с HQ-CLI `host mon.au-team.irpo` и сайт мониторинга; извне доступ запрещен. Повторно проверь `https://web.au-team.irpo`. В отчет: выбор Prometheus/Grafana, порты 80/3000/9090/9100, метрики и ограничение доступа.
