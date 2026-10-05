# Часовой пояс

На всех устройствах, кроме виртуального HQ-SW. Выбери пояс места проведения экзамена; ниже пример для Москвы.

Альт:

```bash
timedatectl set-timezone Europe/Moscow
timedatectl
```

EcoRouter, из `configure terminal`:

```text
ntp timezone utc+3
write memory
```

Проверка из привилегированного режима: `show ntp timezone`.

> **Важно:** часовой пояс и синхронизация времени — разные настройки. Здесь меняется только пояс.
