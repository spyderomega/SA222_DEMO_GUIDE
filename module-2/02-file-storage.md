# RAID0 — HQ-SRV

Нужно: два дополнительных диска по 1 ГБ → `/dev/md0` → раздел ext4 → `/raid` при загрузке.

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
apt-get install -y mdadm parted
```

> **Важно:** `/dev/sdb` и `/dev/sdc` ниже — пример. Выбери именно пустые дополнительные диски, не системный. Создание массива и форматирование стирают данные этих дисков. RAID0 не имеет защиты от отказа диска.

```bash
mdadm --create --verbose /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc
mdadm --detail --scan
cat /proc/mdstat
```

Добавь полученную строку `ARRAY` в `/etc/mdadm.conf` один раз, сохранив остальные настройки. Создай раздел на массиве (в исходном примере этот шаг пропущен):

```bash
parted -s /dev/md0 mklabel msdos
parted -s /dev/md0 mkpart primary ext4 1MiB 100%
lsblk /dev/md0
mkfs.ext4 /dev/md0p1
mkdir -p /raid
blkid /dev/md0p1
```

Если раздел еще не появился, сначала перечитай таблицу через `partprobe /dev/md0`. В `/etc/fstab` добавь строку с UUID **раздела**, полученным из blkid:

```fstab
UUID=<UUID_РАЗДЕЛА> /raid ext4 defaults 0 2
```

```bash
systemctl daemon-reload
mount -av
findmnt /raid
mdadm --detail /dev/md0
df -h /raid
```

Проверь сохранение массива и монтирование после перезагрузки, затем переходи к NFS.
