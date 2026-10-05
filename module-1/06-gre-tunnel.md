# Туннель GRE

HQ-RTR и BR-RTR, EcoRouter. Выбери один вариант туннеля.

## HQ-RTR

```text
interface tunnel.0
description "GRE"
ip address 10.10.10.1/30
ip tunnel 172.16.1.2 172.16.2.2 mode gre
exit
write memory
```

## BR-RTR

```text
interface tunnel.0
description "GRE"
ip address 10.10.10.2/30
ip tunnel 172.16.2.2 172.16.1.2 mode gre
exit
write memory
```

Проверка из привилегированного режима: `show interface tunnel.0`; ping до `10.10.10.2` с HQ и до `10.10.10.1` с BR.

> **Важно:** сначала проверь доступность внешнего адреса второго маршрутизатора (`172.16.2.2` / `172.16.1.2`). В `ip tunnel` сначала свой внешний IP, затем удаленный; это не адреса `10.10.10.x`. Режим должен совпадать на обоих концах.

Адреса, режим и настройки туннеля внеси в отчет.
