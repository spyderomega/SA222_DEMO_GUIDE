# Инвентаризация — Ansible на BR-SRV

Используй работающий инвентарь модуля 2. Нужны только HQ-SRV и HQ-CLI, с доступными SSH и Python.

```bash
lsblk
mount /dev/sr0 /mnt
ls /mnt/playbook
mkdir -p /etc/ansible/PC-INFO
cd /etc/ansible
```

Возьми заготовку с ISO, сохрани как `/etc/ansible/get_hostname_address.yml` и проверь отступы. Исправленный вариант:

```yaml
- name: Collect hostname and IPv4
  hosts: HQ-SRV,HQ-CLI
  gather_facts: true
  tasks:
    - name: Save report on BR-SRV
      ansible.builtin.copy:
        dest: "/etc/ansible/PC-INFO/{{ ansible_hostname }}.yml"
        content: |
          Hostname: {{ ansible_hostname | to_json }}
          IP_Address: {{ ansible_default_ipv4.address | to_json }}
      delegate_to: localhost
```

```bash
ansible-playbook get_hostname_address.yml --syntax-check
ansible-playbook get_hostname_address.yml
ls -l PC-INFO
cat PC-INFO/*.yml
```

Ожидаются два YAML-отчета с реальным именем и IPv4 каждого клиента.

> **Важно:** имена `hosts` должны совпадать с инвентарем, включая регистр. Каталог — **PC-INFO**, не PC_INFO. `delegate_to: localhost` сохраняет файлы на BR-SRV; `gather_facts` собирает данные клиентов. Если нет default IPv4, проверь маршрут по умолчанию на клиенте.
