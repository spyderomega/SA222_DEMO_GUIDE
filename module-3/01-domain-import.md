# Импорт CSV — BR-SRV (контроллер домена)

```bash
lsblk
mount /dev/sr0 /mnt
ls /mnt
cp /mnt/Users.csv /root/Users.csv
file /root/Users.csv
```

Сверь регистр имени CSV на ISO. Если ISO уже смонтирован, используй его текущий путь. На BR-SRV должны быть доступны Python-модули `samba` и `ldb` из установки Samba DC. CSV должен быть UTF-8; при другой кодировке конвертируй через `iconv` с правильным исходным `-f`.

> **Важно:** не используй `UTF-8//IGNORE`: это не удаление диакритики, а потеря некорректных байтов. Проверь заголовок и порядок полей: **first;last;role;phone;ou;street;zip;city;country;pass**. Скрипт ниже рассчитан на заголовок и эти десять колонок, учитывает кавычки CSV. Страна — двухбуквенный код; OU — простое имя без специальных символов DN.

Создай `/root/import.py`:

```python
import csv
import ldb
from samba.auth import system_session
from samba.param import LoadParm
from samba.samdb import SamDB

lp = LoadParm()
lp.load_default()
db = SamDB(session_info=system_session(), lp=lp)
base = str(db.domain_dn())
with open('/root/Users.csv', encoding='utf-8-sig', newline='') as f:
    rows = csv.reader(f, delimiter=';')
    next(rows)  # заголовок; убери эту строку, если заголовка нет
    for line, row in enumerate(rows, 2):
        if not row:
            continue
        if len(row) != 10:
            raise ValueError(f'Строка {line}: нужны 10 полей')
        first, last, role, phone, ou, street, zipcode, city, country, password = row
        first, last, ou = first.strip(), last.strip(), ou.strip()
        if not first or not last or not password:
            raise ValueError(f'Строка {line}: пустое имя/фамилия/пароль')
        username = (first[0] + last).lower()
        if db.search(base=base, scope=ldb.SCOPE_SUBTREE,
                     expression=f'(sAMAccountName={ldb.binary_encode(username)})'):
            raise ValueError(f'{username}: уже существует; проверь дубликат')
        if ou and any(char in ou for char in ',+"\\<>;='):
            raise ValueError(f'Строка {line}: OU требует экранирования DN')
        userou = None
        if ou:
            userou = f'OU={ou}'
            oudn = f'{userou},{base}'
            try:
                db.add({'dn': oudn, 'objectClass': 'organizationalUnit'})
            except ldb.LdbError as error:
                if error.args[0] != ldb.ERR_ENTRY_ALREADY_EXISTS:
                    raise
        db.newuser(username, password, givenname=first, surname=last,
                   jobtitle=role, telephonenumber=phone, userou=userou)
        user = db.search(base=base, scope=ldb.SCOPE_SUBTREE,
                         expression=f'(sAMAccountName={ldb.binary_encode(username)})')[0]
        change = ldb.Message()
        change.dn = user.dn
        for attr, value in {'streetAddress': street, 'postalCode': zipcode,
                            'l': city, 'c': country}.items():
            if value:
                change[attr] = ldb.MessageElement(value, ldb.FLAG_MOD_REPLACE, attr)
        if len(change):
            db.modify(change)
        print(f'Импортирован: {username}')
```

```bash
chmod 600 /root/Users.csv /root/import.py
python3 /root/import.py
samba-tool user list
```

Проверь через `samba-tool user show <ЛОГИН>` имя, фамилию, должность, телефон, OU и адресные атрибуты. Логин примера: первая буква имени + фамилия, нижний регистр; если в задании есть отдельная колонка логина, используй ее.

На HQ-CLI войди под импортированным пользователем с его паролем из CSV. При ошибке проверь DNS DC и время. Для ADMC при необходимости получи билет: `kinit Administrator@AU-TEAM.IRPO`.

> **При ошибке импорта:** скрипт останавливается, уже созданные пользователи сохраняются. Исправь проблемную строку и запускай только оставшиеся, не меняй пароли и не удаляй учетные записи вслепую. Проверь соответствие импорта всем строкам CSV.
