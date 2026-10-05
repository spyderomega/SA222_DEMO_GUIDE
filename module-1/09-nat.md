# NAT офисов — EcoRouter

Настрой в режиме `configure terminal`.

## HQ-RTR

```text
interface isp
ip nat outside
exit
interface vl100
ip nat inside
exit
interface vl200
ip nat inside
exit
interface vl999
ip nat inside
exit
ip nat pool VLAN100 192.168.100.1-192.168.100.30
ip nat pool VLAN200 192.168.200.1-192.168.200.254
ip nat pool VLAN999 192.168.99.1-192.168.99.6
ip nat source dynamic inside-to-outside pool VLAN100 overload interface isp
ip nat source dynamic inside-to-outside pool VLAN200 overload interface isp
ip nat source dynamic inside-to-outside pool VLAN999 overload interface isp
write memory
```

## BR-RTR

```text
interface isp
ip nat outside
exit
interface int1
ip nat inside
exit
ip nat pool BR-Net 192.168.0.1-192.168.0.14
ip nat source dynamic inside-to-outside pool BR-Net overload interface isp
exit
write memory
```

С HQ-SRV и BR-SRV создай трафик: `ping 77.88.8.7`. Затем на маршрутизаторе:

```text
show ip nat translations
```

> **Важно:** `isp` — outside, локальные интерфейсы — inside. Для выхода в Интернет также нужны default route и NAT на ISP. Пустая таблица до появления трафика сама по себе не означает ошибку. Доступ по имени проверяй после DNS.
