# День 2 — Banner Grabbing + python-nmap + JSON отчёт

**Дата:** 10.04.2026  
**Тема:** Углублённый Banner Grabbing, автоматизация nmap, структурированные отчёты  
**Время:** ~6 часов  
**Результаты тестов:** 10/10 · 11/12 · 8/8 · 15/15 · 10/10 · 12/12  

---

## 📖 Теория

### Блок 1 — Banner Grabbing

#### Два типа сервисов

**Активные** — сами отвечают при подключении без запроса:
```
FTP  (21)  → 220 ProFTPD 1.3.5 Server
SSH  (22)  → SSH-2.0-OpenSSH_8.9p1
SMTP (25)  → 220 mail.example.com ESMTP Postfix
POP3 (110) → +OK Dovecot ready
```

**Пассивные** — молчат пока не спросишь:
```python
# HTTP нужно сначала отправить запрос
s.sendall(b'HEAD / HTTP/1.0\r\nHost: target.com\r\n\r\n')
# Только потом отвечает:
# HTTP/1.1 200 OK
# Server: Apache/2.4.7
```

#### Что даёт баннер пентестеру?

```
SSH-2.0-OpenSSH_6.6.1p1 Ubuntu-2ubuntu2.13
│         │               └── ОС: Ubuntu
│         └── Версия ПО: 6.6.1p1 → ищем CVE
└── Протокол: SSH версии 2.0
```

#### HEAD vs GET для HTTP

```
GET  → возвращает заголовки + всё тело страницы (много трафика)
HEAD → возвращает ТОЛЬКО заголовки (быстро, мало трафика)
```
При сканировании всегда используем `HEAD` — нужен только баннер.

#### Таймаут и время сканирования

```
1000 портов × 3 сек таймаут = 3000 сек = 50 минут (без threading)
```
Именно поэтому на День 3 добавим threading — сократим до секунд.

---

### Блок 2 — python-nmap

**python-nmap** запускает nmap из Python и возвращает результаты как словарь:

```
Python скрипт → python-nmap → nmap (subprocess) → XML → словарь Python
```

#### Структура данных

```python
nm[host]['tcp'][22] = {
    'state':   'open',
    'name':    'ssh',
    'product': 'OpenSSH',
    'version': '6.6.1p1',
    'reason':  'syn-ack'
}
```

> ⚠️ Ключ порта — **целое число** (int), не строка!
> `nm[host]['tcp'][22]` ✅  
> `nm[host]['tcp']['22']` ❌ → KeyError

#### Важные аргументы nmap

| Аргумент | Что делает |
|----------|-----------|
| `-sV` | Определяет версии сервисов |
| `--open` | Показывает только открытые порты |
| `-T4` | Скорость сканирования (1-5) |
| `-sS` | SYN scan (нужен root) |

#### tcpwrapped

```
[+] 22    open    tcpwrapped
```
TCP handshake успешен, но сервис сразу закрыл соединение не раскрыв версию.
Наш banner grabber поймал SSH баннер — потому что читал быстрее.

---

### Блок 3 — JSON отчёт

#### Три главные функции модуля json

```python
import json

json.dumps(data)          # словарь → строка (s = string)
json.dump(data, file)     # словарь → файл
json.load(file)           # файл → словарь
```

#### datetime — текущее время

```python
from datetime import datetime
now = datetime.now()
print(now.strftime('%Y-%m-%d %H:%M:%S'))
# → 2026-04-10 07:15:29
```

`%Y` = год · `%m` = месяц · `%d` = день · `%H` = час · `%M` = минуты · `%S` = секунды

#### append() vs =

```python
open_ports = []

open_ports.append({'port': 22})  # ✅ добавляет в список
open_ports.append({'port': 80})
# → [{'port': 22}, {'port': 80}]

open_ports = {'port': 22}  # ❌ заменяет список словарём!
```

---

## 💻 Код — Блок 1: умный Banner Grabber

```python
import socket

# Словарь: порт → что отправить (None = активный сервис)
REQUESTS = {
    21:  None,
    22:  None,
    25:  None,
    80:  b'HEAD / HTTP/1.0\r\nHost: target\r\n\r\n',
    110: None,
    443: b'HEAD / HTTP/1.0\r\nHost: target\r\n\r\n',
}

def grab_banner(ip, port, timeout=2):
    """Подключается к порту и возвращает баннер."""
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(timeout)
        s.connect((ip, port))

        request = REQUESTS.get(port)
        if request:
            s.sendall(request)

        banner = s.recv(1024).decode('utf-8', errors='ignore').strip()
        s.close()
        return banner

    except ConnectionRefusedError:
        return 'CLOSED'
    except socket.timeout:
        return 'FILTERED'
    except Exception as e:
        return f'ERROR: {e}'


# --- главная программа ---
TARGET = '45.33.32.156'

print(f'Сканирование {TARGET}')
print('=' * 50)

for port, _ in REQUESTS.items():
    result = grab_banner(TARGET, port)

    if result == 'CLOSED':
        print(f'[-] {port:<5} CLOSED  |')
    elif result == 'FILTERED':
        print(f'[?] {port:<5} FILTER  |')
    elif result.startswith('ERROR'):
        print(f'[!] {port:<5} ERROR   | {result}')
    else:
        first_line = result.split('\n')[0]
        print(f'[+] {port:<5} OPEN    | {first_line[:55]}')

print('=' * 50)
```

**Результат:**
```
[?] 21    FILTER  |
[+] 22    OPEN    | SSH-2.0-OpenSSH_6.6.1p1 Ubuntu-2ubuntu2.13
[?] 25    FILTER  |
[+] 80    OPEN    | HTTP/1.1 200 OK
[?] 110   FILTER  |
[?] 443   FILTER  |
```

---

## 💻 Код — Блок 2: python-nmap сканер

```python
import nmap

TARGET = '45.33.32.156'
PORTS  = '21-25,80,110,443,3306,8080'

nm = nmap.PortScanner()

print(f'Сканирование {TARGET} через nmap...')
print('=' * 55)

nm.scan(TARGET, PORTS, arguments='-sV --open')

for host in nm.all_hosts():
    print(f'Хост: {host} | Статус: {nm[host].state()}')
    print('-' * 55)

    if 'tcp' in nm[host]:
        for port, data in nm[host]['tcp'].items():
            state   = data['state']
            name    = data['name']
            product = data['product']
            version = data['version']

            print(f'[+] {port:<6} {state:<10} {name:<10} {product} {version}')

print('=' * 55)
print('Сканирование завершено!')
```

**Результат:**
```
[+] 22     open       ssh        OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13
[+] 80     open       http       Apache httpd 2.4.7
```

---

## 💻 Код — Блок 3: финальный скрипт с JSON отчётом

```python
import nmap
import json
from datetime import datetime

# --- конфигурация ---
TARGET = '45.33.32.156'
PORTS  = '21-25,80,110,443,3306,8080'

# --- структура отчёта ---
report = {
    'target':     TARGET,
    'scan_date':  datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
    'open_ports': [],
    'total_open': 0
}

# --- сканирование ---
nm = nmap.PortScanner()
print(f'Сканирование {TARGET}...')
nm.scan(TARGET, PORTS, arguments='-sV --open')

# --- сбор результатов ---
for host in nm.all_hosts():
    if 'tcp' in nm[host]:
        for port, data in nm[host]['tcp'].items():

            port_info = {
                'port':    port,
                'state':   data['state'],
                'name':    data['name'],
                'product': data['product'],
                'version': data['version']
            }

            report['open_ports'].append(port_info)
            report['total_open'] += 1

            print(f"[+] {port:<6} {data['name']:<10} {data['product']} {data['version']}")

# --- сохранение в JSON ---
filename = f"report_{TARGET}_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"

with open(filename, 'w') as f:
    json.dump(report, f, indent=4)

print(f'\nОткрытых портов: {report["total_open"]}')
print(f'Отчёт сохранён: {filename}')
```

**Результат в терминале:**
```
[+] 22     ssh        OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13
[+] 80     http       Apache httpd 2.4.7

Открытых портов: 2
Отчёт сохранён: report_45.33.32.156_20260410_071540.json
```

**Содержимое JSON файла:**
```json
{
    "target": "45.33.32.156",
    "scan_date": "2026-04-10 07:15:29",
    "open_ports": [
        {
            "port": 22,
            "state": "open",
            "name": "ssh",
            "product": "OpenSSH",
            "version": "6.6.1p1"
        },
        {
            "port": 80,
            "state": "open",
            "name": "http",
            "product": "Apache httpd",
            "version": "2.4.7"
        }
    ],
    "total_open": 2
}
```

---

## 🔑 Ключевые выводы дня

1. **Активные сервисы** (SSH, FTP, SMTP) сами отвечают — просто подключись и читай. **Пассивные** (HTTP) молчат — нужно сначала отправить запрос.

2. **Словарь REQUESTS** — чистый способ хранить логику "что отправить на какой порт". Добавить новый порт = одна строка.

3. **python-nmap** возвращает данные как Python словарь — не нужно парсить текст. Ключ порта — `int`, не строка!

4. **tcpwrapped** = порт открыт, но сервис отказался раскрыть версию. Наш banner grabber иногда обходит это.

5. **json.dump() vs json.dumps()** — "s" в конце = string (строка). Без "s" = записывает в файл.

6. **append() vs =** — `list.append(item)` добавляет элемент. `list = item` заменяет весь список.

7. **with open()** — автоматически закрывает файл даже при ошибке. Всегда используй with для файлов.

8. **datetime** нельзя записать в JSON напрямую — нужно конвертировать в строку через `strftime()`.

---

## ✅ Чек-лист дня

- [x] Понял разницу между активными и пассивными сервисами
- [x] Написал banner grabber со словарём запросов и функцией
- [x] Подключил python-nmap и получил версии сервисов
- [x] Понял структуру nm[host]['tcp'][port] (ключ = int!)
- [x] Создал JSON отчёт с датой сканирования
- [x] Разобрался с dump/dumps, append, with open
- [x] Сравнил результаты banner grabber и nmap

---

## 📚 Ресурсы

- [Python socket документация](https://docs.python.org/3/library/socket.html)
- [python-nmap документация](https://xael.org/pages/python-nmap-en.html)
- [Python json документация](https://docs.python.org/3/library/json.html)
- [scanme.nmap.org](http://scanme.nmap.org) — легальный сервер для тестов
- Книга: **Black Hat Python** — Justin Seitz (глава 2-3)

---

*День 2 из 50 завершён ✅ | Следующий: День 3 — argparse CLI*
