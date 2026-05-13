# День 11 — Port Scanner с баннерами 🔍

## Что изучили
- Профессиональный сканер портов с banner grabbing
- `connect_ex()` — проверка порта без исключений
- `settimeout()` — таймаут для сокета
- Параллельное сканирование через `ThreadPoolExecutor`
- Сохранение результатов в JSON

---

## Итоговый код — day11.py

```python
import socket, argparse, json, threading
from concurrent.futures import ThreadPoolExecutor

parser = argparse.ArgumentParser()
parser.add_argument('host')
parser.add_argument('--ports', nargs='+', type=int, default=[21,22,23,80,443,8080])
args = parser.parse_args()
lock = threading.Lock()
results = []

def scan(port):
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(1)
        result = s.connect_ex((args.host, port))
        if result == 0:
            banner = ""
            try:
                banner = s.recv(1024).decode().strip()
            except:
                pass
            with lock:
                print(f"[+] {port} OPEN | {banner}")
                results.append({"port": port, "banner": banner})
        s.close()
    except:
        pass

print("Начинаем Сканирование")
with ThreadPoolExecutor(max_workers=10) as executor:
    executor.map(scan, args.ports)

with open(f"{args.host}_scan.json", 'w') as f:
    json.dump(results, f, indent=4)
```

---

## Как запустить

```bash
# Сканировать дефолтные порты
python3 day11.py scanme.nmap.org

# Сканировать конкретные порты
python3 day11.py 127.0.0.1 --ports 22 80 443 8080

# Результат сохранится в файл
cat 127.0.0.1_scan.json
```

---

## Пример вывода

```
Начинаем Сканирование
[+] 22 OPEN | SSH-2.0-OpenSSH_8.9p1
[+] 80 OPEN |
[+] 8080 OPEN |
```

---

## Ключевые концепции

### connect() vs connect_ex()
```python
# connect() — бросает исключение если порт закрыт
s.connect((host, port))   # → exception если закрыт

# connect_ex() — возвращает код ошибки
result = s.connect_ex((host, port))
if result == 0:           # → 0 = открыт
    print("OPEN")         # → другое число = закрыт
```

### settimeout() — таймаут сокета
```python
s.settimeout(1)  # ждём максимум 1 секунду
                 # если нет ответа — идём дальше
```

### Откуда берётся баннер?
```
SSH:    SSH-2.0-OpenSSH_8.9p1   ← приветствует сразу
FTP:    220 FTP Server ready     ← приветствует сразу
HTTP:   (нет баннера)            ← нужен запрос
Flask:  (нет баннера)            ← нужен запрос
```

### Зачем s.close()?
```python
s.close()  # закрываем сокет после проверки
           # освобождаем системные ресурсы
           # без этого → утечка памяти
```

---

## Дефолтные порты и что на них живёт

| Порт | Сервис |
|---|---|
| `21` | FTP — передача файлов |
| `22` | SSH — удалённый доступ |
| `23` | Telnet — старый удалённый доступ |
| `80` | HTTP — веб сайт |
| `443` | HTTPS — защищённый веб |
| `8080` | HTTP альтернативный |

---

## Прогресс курса

| День | Тема | Статус |
|---|---|---|
| День 1 | Сокеты, TCP клиент, banner grabbing | ✅ |
| День 2 | JSON отчёты | ✅ |
| День 3 | argparse CLI | ✅ |
| День 4 | Threading, ThreadPoolExecutor | ✅ |
| День 5 | SSH Brute Force (paramiko) | ✅ |
| День 6 | HTTP разведка с requests | ✅ |
| День 7 | Subdomain Enumeration | ✅ |
| День 8 | HTTP Login Brute Force | ✅ |
| День 9 | DNS Разведка | ✅ |
| День 10 | WHOIS Разведка | ✅ |
| День 11 | Port Scanner с баннерами | ✅ |

---

*Python для Offensive Security — 50 дней*
