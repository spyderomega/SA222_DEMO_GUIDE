# Ansible — BR-SRV

## Установка

```bash
apt-get update
apt-get install -y ansible sshpass python3-module-pip
mkdir -p /etc/ansible
ansible-galaxy collection install ansible.netcommon cisco.ios
pip3 install ansible-pylibssh
```

> **Важно:** если pip запрещает установку в системную среду, используй пакет библиотеки из репозитория или venv с Ansible и библиотекой в одной среде. Не скрывай эту ошибку настройками предупреждений.

## Инвентарь `/etc/ansible/hosts`

```ini
[linux]
HQ-SRV ansible_host=192.168.100.2 ansible_user=sshuser ansible_port=2026
HQ-CLI ansible_host=192.168.200.2 ansible_user=<ЛОКАЛЬНЫЙ_ПОЛЬЗОВАТЕЛЬ_HQ-CLI> ansible_port=22

[linux:vars]
ansible_python_interpreter=/usr/bin/python3

[routers]
HQ-RTR ansible_host=10.10.10.1
BR-RTR ansible_host=192.168.0.1

[routers:vars]
ansible_user=net_admin
ansible_password=P@ssw0rd
ansible_connection=ansible.netcommon.network_cli
ansible_network_os=cisco.ios.ios
```

Для Linux добавь к каждой строке `ansible_password=<ПАРОЛЬ_ЭТОГО_ПОЛЬЗОВАТЕЛЯ>` или заранее настрой SSH-ключ. Для HQ-SRV пароль примера — `P@ssw0rd`; учетную запись и пароль HQ-CLI проверь отдельно. Проверь IP HQ-CLI после DHCP/ввода в домен и доступность SSH/Python на Linux.

`/etc/ansible/ansible.cfg`:

```ini
[defaults]
inventory = /etc/ansible/hosts
host_key_checking = False
```

На обоих EcoRouter из `configure terminal`:

```text
security none
write memory
```

> **Учебный стенд:** `security none` и отключенная проверка SSH host key — упрощения исходного практикума. `cisco.ios.ios` используется как совместимый драйвер CLI; работоспособность зависит от версии EcoRouter/коллекций.

## Проверка с BR-SRV

```bash
cd /etc/ansible
ansible-inventory --graph
ansible all -m ping
ansible routers -m ansible.netcommon.cli_command -a "command=show hostname"
```

По заданию все четыре узла должны ответить `pong` на `ansible all -m ping` без ошибок и предупреждений.

> **Важно:** для Linux `ping` проверяет SSH и Python. На `network_cli` ответ pong сам по себе не доказывает доступ к маршрутизатору; обязательно проверь реальную CLI-команду. При ошибке начни с обычного SSH на нужный порт, затем используй `-vvv`. [Документация Ansible ping](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/ping_module.html).
