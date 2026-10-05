# ГОСТ-сертификаты и HTTPS

## 1. ЦС и сертификаты — HQ-SRV

```bash
apt-get install -y openssl-gost-engine
control openssl-gost enabled
openssl ciphers | tr ":" "\n" | grep GOST
mkdir -p /root/ca
cd /root/ca
umask 077
openssl genpkey -algorithm gost2012_256 -pkeyopt paramset:TCB -out ca.key
openssl req -new -x509 -md_gost12_256 -days 30 -key ca.key -out ca.crt -subj "/CN=AU-TEAM CA" -addext "basicConstraints=critical,CA:TRUE" -addext "keyUsage=critical,keyCertSign,cRLSign"
```

Для каждого имени создай ключ, запрос и сертификат на 30 дней с SAN:

```bash
for name in web.au-team.irpo docker.au-team.irpo; do
    openssl genpkey -algorithm gost2012_256 -pkeyopt paramset:A -out "$name.key"
    openssl req -new -md_gost12_256 -key "$name.key" -out "$name.csr" -subj "/CN=$name"
    printf 'subjectAltName=DNS:%s\nbasicConstraints=CA:FALSE\nkeyUsage=digitalSignature\nextendedKeyUsage=serverAuth\n' "$name" > "$name.ext"
    openssl x509 -req -md_gost12_256 -in "$name.csr" -CA ca.crt -CAkey ca.key -CAcreateserial -out "$name.crt" -days 30 -extfile "$name.ext"
    openssl verify -CAfile ca.crt "$name.crt"
    openssl x509 -in "$name.crt" -noout -dates -ext subjectAltName
done
```

> **Важно:** не пересоздавай работающий ЦС при повторном запуске. SAN должен содержать точное имя сайта, время на всех узлах должно совпадать. Закрытый **ca.key остается на HQ-SRV**; на ISP передаются только ключи и сертификаты сайтов.

```bash
scp web.au-team.irpo.key web.au-team.irpo.crt docker.au-team.irpo.key docker.au-team.irpo.crt root@172.16.1.1:/etc/nginx/
```

Если root по SSH запрещен, скопируй через доступную учетную запись во временный каталог ISP, затем перенеси от root.

## 2. HTTPS — ISP

Установи `openssl-gost-engine`, выполни `control openssl-gost enabled`, проверь доступность нужного шифра через `openssl ciphers`. В существующих server-блоках nginx для **обоих** сайтов замени `listen 80` на `listen 443 ssl` и добавь:

```nginx
ssl_certificate /etc/nginx/web.au-team.irpo.crt;
ssl_certificate_key /etc/nginx/web.au-team.irpo.key;
ssl_ciphers GOST2012-KUZNYECHIK-KUZNYECHIKOMAC;
ssl_protocols TLSv1.2;
ssl_prefer_server_ciphers on;
```

В блоке docker пути — `docker.au-team.irpo.crt` и `.key`. Сохрани `server_name`, `proxy_pass`, proxy-заголовки и аутентификацию WEB для web. Для перенаправления HTTP добавь отдельный блок:

```nginx
server {
    listen 80;
    server_name web.au-team.irpo docker.au-team.irpo;
    return 301 https://$host$request_uri;
}
```

```bash
chmod 600 /etc/nginx/web.au-team.irpo.key /etc/nginx/docker.au-team.irpo.key
nginx -t
systemctl restart nginx
```

> **Если `no cipher match` или ключ не читается:** проверь ГОСТ-поддержку OpenSSL и сборки nginx, доступные cipher suites и права. Не заменяй ГОСТ обычным шифром ради успешного запуска. [Поддержка ГОСТ в Альт](https://www.altlinux.org/ГОСТ_в_OpenSSL).

## 3. Доверие — HQ-CLI

Передай **ca.crt** на HQ-CLI, затем от root:

```bash
cp /tmp/ca.crt /etc/pki/ca-trust/source/anchors/au-team-ca.crt
update-ca-trust
apt-get install -y openssl-gost-engine
control openssl-gost enabled
```

Распакуй дистрибутив КриптоПро CSP 5 с GUI и запусти из каталога распаковки `bash linux-amd64/install_gui.sh`. Выбери копирование корневых сертификатов. В «Инструментах работы с криптографией → Сертификаты» проверь доверие ЦС; если его нет, импортируй ca.crt в доверенные корневые сертификаты пользователя браузера.

В Яндекс Браузере разреши использование ГОСТ и открой оба **https**-адреса. Предупреждений о недоверенном сертификате/имени быть не должно; web сохраняет вход `WEB / P@ssw0rd`.

Проверка на HQ-CLI:

```bash
openssl s_client -connect web.au-team.irpo:443 -servername web.au-team.irpo -CAfile /etc/pki/ca-trust/source/anchors/au-team-ca.crt -verify_hostname web.au-team.irpo -verify_return_error -tls1_2
```

Повтори для docker. Ожидаются ГОСТ-шифр и успешная проверка сертификата. TLS завершается на ISP; upstream остается HTTP через проброшенные порты.
