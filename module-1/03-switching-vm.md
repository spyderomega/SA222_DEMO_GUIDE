# Коммутация: отдельная ВМ HQ-SW

Пример портов: `ens3` → HQ-RTR (trunk), `ens4` → HQ-SRV (VLAN 100), `ens5` → HQ-CLI (VLAN 200). Сверь по MAC.

```bash
apt-get install -y openvswitch
systemctl enable --now openvswitch
```

Для каждого порта создай `/etc/net/ifaces/<ИНТЕРФЕЙС>/options`: `TYPE=eth`; интерфейсы должны быть включены и переведены в manual.

```bash
systemctl restart network
ovs-vsctl add-br SW
ovs-vsctl add-port SW ens3 trunks=100,200,999
ovs-vsctl add-port SW ens4 tag=100
ovs-vsctl add-port SW ens5 tag=200
ovs-vsctl show
```

> **Важно:** на HQ-RTR VLAN 100/200/999 идут через один порт. HQ-SRV и HQ-CLI получают нетегированный трафик, VLAN задает HQ-SW. В исходнике `trunk` исправлен на `trunks`. При повторном выполнении сначала проверь существующий мост и порты.

Проверь с HQ-SRV `ping 192.168.100.1`; сведения о коммутации внеси в отчет.
