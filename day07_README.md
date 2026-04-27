# День 7 — Subdomain Enumeration 🔍

## Что изучили
- DNS резолвинг через `socket.gethostbyname()`
- Перебор поддоменов по wordlist
- Обработка ошибок через `try/except`
- Параллельное сканирование через `ThreadPoolExecutor`

---

## Итоговый код — day7.py

```python
import socket, requests, argparse, json, threading
from concurrent.futures import ThreadPoolExecutor

parser = argparse.ArgumentParser()
parser.add_argument('domain')
parser.add_argument('--wordlist', default='subdomains.txt')
args = parser.parse_args()
lock = threading.Lock()
results = []

with open(args.wordlist, 'r') as f:
    subdomains = f.read().splitlines()

def scan(subdomain):
    try:
        ip = socket.gethostbyname(f"{subdomain}.{args.domain}")
        url = f"http://{subdomain}.{args.domain}"
        response = requests.get(url, timeout=3)
        print(f"[+] {subdomain}.{args.domain} - IP: {ip} - {response.status_code}")
    except:
        pass

with ThreadPoolExecutor(max_workers=10) as executor:
    executor.map(scan, subdomains)
```

---

## Как запустить

```bash
# Создать wordlist поддоменов
echo -e "www\nmail\nadmin\ndev\nstaging\napi\nftp\ntest\nportal\nblog" > subdomains.txt

# Запустить сканер
python3 day7.py google.com

# С кастомным wordlist
python3 day7.py google.com --wordlist mylist.txt
```

---

## Пример вывода

```
[+] www.google.com - IP: 142.251.151.119 - 200
[+] api.google.com - IP: 142.251.20.103 - 404
[+] blog.google.com - IP: 142.251.20.191 - 200
[+] mail.google.com - IP: 142.251.13.18 - 200
[+] admin.google.com - IP: 142.251.14.113 - 200
```

---

## Ключевые концепции

### DNS резолвинг
```python
# Берёт домен → возвращает IP адрес
ip = socket.gethostbyname("www.google.com")
# → '142.251.151.119'
```

### try/except — обработка ошибок
```python
try:
    # Пробуем выполнить код
    ip = socket.gethostbyname(f"{subdomain}.{args.domain}")
except:
    pass  # Поддомен не существует — пропускаем
```

### Что означают результаты
| Результат | Значение |
|---|---|
| `[+] www.google.com - IP: x.x.x.x - 200` | Поддомен найден! ✅ |
| `[+] api.google.com - IP: x.x.x.x - 404` | DNS есть, страница не найдена |
| Нет вывода | Поддомен не существует в DNS |

### Ошибки DNS
| Ошибка | Значение |
|---|---|
| `Name or service not known` | Поддомен не существует в DNS |
| `No address associated with hostname` | DNS не вернул IP адрес |

---

## Новые команды

```python
# DNS резолвинг
ip = socket.gethostbyname(f"{subdomain}.{args.domain}")

# Обработка ошибок
try:
    # код
except:
    pass

# Отладка ошибок
except Exception as e:
    print(f"[-] {subdomain} — {e}")
```

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

---

*Python для Offensive Security — 50 дней*
