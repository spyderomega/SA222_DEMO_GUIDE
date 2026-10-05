# NFS — HQ-SRV → HQ-CLI

## HQ-SRV

Сначала убедись, что RAID смонтирован в `/raid`:

```bash
findmnt /raid
apt-get install -y nfs-server nfs-utils
mkdir -p /raid/nfs
chmod 777 /raid/nfs
```

В `/etc/exports` добавь:

```text
/raid/nfs 192.168.200.0/24(rw,sync,no_subtree_check,no_root_squash)
```

```bash
systemctl enable --now nfs-server
exportfs -rav
exportfs -v
```

> **Важно:** доступ разрешен только сети HQ-CLI; сверь ее префикс со стендом. `chmod 777` и `no_root_squash` — настройки учебного примера: любой пользователь клиента может писать, root клиента получает права root на ресурсе.

## HQ-CLI

```bash
apt-get install -y nfs-utils nfs-clients
mkdir -p /mnt/nfs
```

В `/etc/fstab` добавь:

```fstab
192.168.100.2:/raid/nfs /mnt/nfs nfs defaults,_netdev 0 0
```

```bash
systemctl daemon-reload
mount -av
findmnt /mnt/nfs
touch /mnt/nfs/check-hq-cli
```

На HQ-SRV: `ls -l /raid/nfs/check-hq-cli`. Проверь монтирование после перезагрузки HQ-CLI.

> **Если не монтируется:** проверь маршрут до HQ-SRV, службу NFS, exports и firewall. Если `/raid` не смонтирован, остановись: иначе файлы попадут на системный диск.

В отчет: путь ресурса, разрешенная сеть, параметры exports и монтирование клиента.
