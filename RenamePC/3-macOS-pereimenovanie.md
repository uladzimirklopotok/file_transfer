# 🍏 Как переименовать компьютер в macOS

---

## Способ 1 — через «Системные настройки»

1. Откройте **Системные настройки** → **Основные** → **Общий доступ** (*System Settings → General → Sharing*).
2. В поле **Имя компьютера** введите новое название.
3. Закройте окно — изменения применятся автоматически.

---

## Способ 2 — через Терминал

```bash
# Отображаемое имя компьютера
sudo scutil --set ComputerName "Новое-Имя"

# Имя для локальной сети (Bonjour)
sudo scutil --set LocalHostName "Novoe-Imya"

# Имя хоста
sudo scutil --set HostName "Novoe-Imya"
```

Проверить текущие имена:

```bash
scutil --get ComputerName
scutil --get LocalHostName
scutil --get HostName
```

---

> 💡 **Совет:** `LocalHostName` не может содержать пробелы — используйте дефис вместо них.
