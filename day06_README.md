# День 6 — HTTP разведка с requests 🌐

## Что изучили
- Отправка HTTP запросов с помощью `requests`
- Чтение и анализ HTTP заголовков (fingerprinting)
- Directory fuzzing — перебор путей на сервере
- Параллельный фаззинг через `ThreadPoolExecutor`

---

## Итоговый код — day6.py

```python
import requests, argparse, threading
from concurrent.futures import ThreadPoolExecutor

parser = argparse.ArgumentParser(description='Port Scanner')
parser.add_argument('host')
parser.add_argument('--wordlist', default='paths.txt')
args = parser.parse_args()
lock = threading.Lock()

with open(args.wordlist, 'r') as f:
    paths = f.read().splitlines()

url = f"http://{args.host}"
response = requests.get(url, timeout=3)
print(f"\n[*] Заголовки {args.host}:")
for key, value in response.headers.items():
    print(f"  {key}: {value}")

def fuzz(path):
    url = f"http://{args.host}/{path}"
    response = requests.get(url, timeout=3)
    if response.status_code != 404:
        print(f'[+] {response.status_code} - /{path}')

with ThreadPoolExecutor(max_workers=10) as executor:
    executor.map(fuzz, paths)
```

---

## Как запустить

```bash
# Создать wordlist
echo -e "admin\nlogin\nindex\nbackup\nconfig\ntest\nuploads\nphpmyadmin" > paths.txt

# Запустить сканер
python3 day6.py scanme.nmap.org

# С кастомным wordlist
python3 day6.py scanme.nmap.org --wordlist mylist.txt
```

---

## Пример вывода

```
[*] Заголовки scanme.nmap.org:
  Date: Sat, 25 Apr 2026 06:01:14 GMT
  Server: Apache/2.4.7 (Ubuntu)
  Accept-Ranges: bytes
  Content-Encoding: gzip
  Content-Type: text/html

[+] 200 - /index
[+] 403 - /admin
```

---

## Ключевые концепции

### HTTP заголовки для разведки
| Заголовок | Что раскрывает |
|---|---|
| `Server` | Тип и версия веб-сервера |
| `X-Powered-By` | Язык/фреймворк (PHP, ASP.NET) |
| `Set-Cookie` | Технологии сессий |
| `Content-Security-Policy` | Уровень защиты сервера |

### Коды ответов HTTP
| Код | Значение | Важность |
|---|---|---|
| `200` | Страница существует | 🎯 Найдено! |
| `403` | Доступ запрещён | ⚠️ Существует, но закрыто |
| `404` | Не найдено | ❌ Пропускаем |

### Fingerprinting — что ищем
```
Server: Apache/2.4.7 (Ubuntu)  ← версия + ОС = ищем CVE
Server: gws                     ← Google скрывает версию ✅
X-Frame-Options: SAMEORIGIN     ← защита от Clickjacking ✅
Set-Cookie: Secure; HttpOnly    ← защищённые куки ✅
```

---

## Новые команды

```python
# HTTP GET запрос
response = requests.get(url, timeout=3)

# Чтение заголовков
for key, value in response.headers.items():
    print(f"  {key}: {value}")

# Проверка кода ответа
if response.status_code != 404:
    print(f"[+] {response.status_code} - /{path}")

# Параллельный фаззинг
with ThreadPoolExecutor(max_workers=10) as executor:
    executor.map(fuzz, paths)
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

---

*Python для Offensive Security — 50 дней*
