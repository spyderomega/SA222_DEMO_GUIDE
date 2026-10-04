# Настройка ipsec на EcoRouter

**Печатные страницы:** 135–136.

## Где изучается?

4 курс:

– Безопасность компьютерных сетей;

– Программное обеспечение компьютерных сетей.



Настройка ipsec на EcoRouter



## Подробное описание пункта задания

Перенастройте ip-туннель с базового до уровня туннеля, обеспечивающего

шифрование трафика:

• настройте защищенный туннель между HQ-RTR и BR-RTR;

• внесите необходимые изменения в конфигурацию динамической маршрутизации, протокол динамической маршрутизации должен возобновить

работу после перенастройки туннеля;

• выбранное программное обеспечение, обоснование его выбора и его основные параметры, изменения в конфигурации динамической маршрутизации

отметьте в отчете.

## Как делать?

Описать профиль, криптокарту и криптофильтр, затем применить криптокарту к криптофильтру, затем применить криптофильтр к туннелю. Рекомен-

дуется использовать максимальный размер пакета равный 1360 для работы

сети поверх ipsec.

Конфигурацию HQ-RTR и BR-RTR дополнить в соответствии с таблицей,

в таблице указаны параметры, что нужно дополнить, без учета предыдущей

настройки:



HQ-RTR

crypto-ipsec ike enable

!

crypto-ipsec profile IPSEC ike-v2

mode tunnel

nat-traversal

ike-phase1

proposal aes256-sha256-modp2048

auth pre-shared-key P@ssw0rd

ike-phase2

protocol esp

proposal aes256-sha256

local-ts 172.16.1.2

remote-ts 172.16.2.2

!

crypto-map CMAP 10

match peer 172.16.2.2

set crypto-ipsec profile IPSEC

!



КОД 09.02.06-1-2026 СЕТЕВОЙ И СИСТЕМНЫЙ АДМИНИСТРАТОР



filter-map ipv4 FMAP 10

match gre host 172.16.1.2 host 172.16.2.2

set crypto-map CMAP peer 172.16.2.2

!

filter-map ipv4 FMAP 20

match udp host 172.16.2.2 eq 4500 host 172.16.1.2 eq 4500

set crypto-map CMAP peer 172.16.2.2

!

filter-map ipv4 FMAP 30

match any any any

set accept

!

interface tunnel.0

ip mtu 1360

set filter-map in FMAP 10



Выполнить те же действия, но с другой стороны, указав нужные параметры

на BR-RTR:



BR-RTR

crypto-ipsec ike enable

!

crypto-ipsec profile IPSEC ike-v2

mode tunnel

nat-traversal

ike-phase1

proposal aes256-sha256-modp2048

auth pre-shared-key P@ssw0rd

ike-phase2

protocol esp

proposal aes256-sha256

local-ts 172.16.1.2

remote-ts 172.16.2.2

!

crypto-map CMAP 10

match peer 172.16.2.2

set crypto-ipsec profile IPSEC

!

filter-map ipv4 FMAP 10

match gre host 172.16.1.2 host 172.16.2.2

set crypto-map CMAP peer 172.16.2.2

!

filter-map ipv4 FMAP 20

match udp host 172.16.2.2 eq 4500 host 172.16.1.2 eq 4500

set crypto-map CMAP peer 172.16.2.2

!

filter-map ipv4 FMAP 30

match any any any
