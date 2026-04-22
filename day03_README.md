# День 3 — argparse: профессиональный CLI

**Дата:** 10.04.2026  
**Тема:** argparse — аргументы командной строки  
**Тесты:** Теория 10/10 · Код 10/10  

---

## 📖 Теория

### Зачем нужен argparse?

Без argparse — нужно лезть в код чтобы изменить параметры:
```python
TARGET = '45.33.32.156'  # меняем вручную каждый раз
```

С argparse — всё через терминал:
```bash
python3 scanner.py --host 45.33.32.156 --ports 22,80,443 --output report.json
```

---

### Три вида аргументов

| Тип | Пример | Обязательный? |
|-----|--------|--------------|
| Позиционный | `python3 tool.py 45.33.32.156` | Да |
| Опциональный | `python3 tool.py --ports 22,80` | Нет |
| Флаг | `python3 tool.py --verbose` | Нет |

---

### Три шага работы с argparse

**Шаг 1 — Создаём парсер:**
```python
import argparse

parser = argparse.ArgumentParser(
    description='Мой сканер портов'
)
```

**Шаг 2 — Добавляем аргументы:**
```python
# Позиционный (обязательный):
parser.add_argument('host', help='IP адрес цели')

# Опциональный с коротким именем:
parser.add_argument('-p', '--ports',
    default='22,80,443',
    help='Порты через запятую')

# С типом данных:
parser.add_argument('-t', '--timeout',
    type=float, default=2.0,
    help='Таймаут в секундах')

# Флаг True/False:
parser.add_argument('-v', '--verbose',
    action='store_true',
    help='Подробный вывод')

# Сохранение в файл:
parser.add_argument('-o', '--output',
    help='Файл для отчёта')
```

**Шаг 3 — Читаем аргументы:**
```python
args = parser.parse_args()

print(args.host)     # '45.33.32.156'
print(args.ports)    # '22,80,443'
print(args.timeout)  # 2.0
print(args.verbose)  # True или False
print(args.output)   # 'report.json' или None
```

---

### Важные параметры add_argument

| Параметр | Что делает |
|---------|-----------|
| `type=int` | Конвертирует строку в int автоматически |
| `default=` | Значение если аргумент не указан |
| `help=` | Описание в --help |
| `required=True` | Делает опциональный аргумент обязательным |
| `choices=[]` | Ограничивает допустимые значения |
| `action='store_true'` | Создаёт флаг True/False |

---

### --help генерируется автоматически

```bash
$ python3 scanner.py --help

usage: scanner.py [-h] [-p PORTS] [-t TIMEOUT] [-v] [-o OUTPUT] host

Мой сканер портов

positional arguments:
  host                  IP адрес цели

options:
  -h, --help            show this help message and exit
  -p, --ports PORTS     Порты через запятую
  -t, --timeout TIMEOUT Таймаут в секундах
  -v, --verbose         Подробный вывод
  -o, --output OUTPUT   Файл для отчёта
```

---

## 💻 Финальный код — day3.py

```python
import argparse
import socket
import json
from datetime import datetime

# --- аргументы CLI ---
parser = argparse.ArgumentParser(
    description='Python Port Scanner — День 3'
)

parser.add_argument('host',
    help='IP адрес или hostname цели')

parser.add_argument('-p', '--ports',
    default='21,22,25,80,443,3306,8080',
    help='Порты через запятую (по умолчанию: топ-7)')

parser.add_argument('-t', '--timeout',
    type=float, default=2.0,
    help='Таймаут соединения в секундах')

parser.add_argument('-v', '--verbose',
    action='store_true',
    help='Показывать закрытые и фильтруемые порты')

parser.add_argument('-o', '--output',
    help='Сохранить отчёт в JSON файл')

args = parser.parse_args()

# --- разбираем порты из строки ---
ports = [int(p.strip()) for p in args.ports.split(',')]

# --- функция сканирования ---
def scan_port(host, port, timeout):
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(timeout)
        s.connect((host, port))

        if port in [80, 8080, 443]:
            s.sendall(b'HEAD / HTTP/1.0\r\nHost: target\r\n\r\n')

        banner = s.recv(1024).decode('utf-8', errors='ignore').strip()
        s.close()
        return 'OPEN', banner.split('\n')[0][:55]

    except ConnectionRefusedError:
        return 'CLOSED', ''
    except socket.timeout:
        return 'FILTER', ''
    except Exception as e:
        return 'ERROR', str(e)


# --- главная программа ---
print(f'Сканирование: {args.host}')
print(f'Портов:       {len(ports)}')
print(f'Таймаут:      {args.timeout}s')
print('=' * 55)

open_ports = []
open_count = 0

for port in ports:
    status, banner = scan_port(args.host, port, args.timeout)

    if status == 'OPEN':
        print(f'[+] {port:<6} OPEN   | {banner}')
        open_ports.append({'port': port, 'banner': banner})
        open_count += 1
    elif args.verbose:
        symbol = '[-]' if status == 'CLOSED' else '[?]'
        print(f'{symbol} {port:<6} {status:<7}|')

print('=' * 55)
print(f'Открытых портов: {open_count}')

# --- сохранение в JSON если указан --output ---
if args.output:
    report = {
        'host':       args.host,
        'scan_date':  datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
        'timeout':    args.timeout,
        'open_ports': open_ports,
        'total_open': open_count
    }
    with open(args.output, 'w') as f:
        json.dump(report, f, indent=4)
    print(f'Отчёт сохранён: {args.output}')
```

---

## 🚀 Примеры запуска

```bash
# Справка (автогенерированная):
python3 day3.py --help

# Базовый запуск — только host обязателен:
python3 day3.py 45.33.32.156

# Кастомные порты:
python3 day3.py 45.33.32.156 -p 22,80,443

# Подробный вывод с сохранением:
python3 day3.py 45.33.32.156 -p 22,80 -v -o report.json

# Быстрый таймаут:
python3 day3.py 45.33.32.156 -t 1.0 -p 22,80
```

---

## 📊 Результат

```
Сканирование: 45.33.32.156
Портов:       2
Таймаут:      2.0s
═══════════════════════════════════════════════════════
[+] 22     OPEN   | SSH-2.0-OpenSSH_6.6.1p1 Ubuntu-2ubuntu2.13
[+] 80     OPEN   | HTTP/1.1 200 OK
═══════════════════════════════════════════════════════
Открытых портов: 2
Отчёт сохранён: report.json
```

---

## 🔑 Ключевые выводы дня

1. **argparse** превращает скрипт в профессиональный инструмент с CLI.

2. **Позиционный аргумент** — обязательный, без имени. **Опциональный** — начинается с `--`, необязательный.

3. **action='store_true'** создаёт флаг — `--verbose` без значения → True/False.

4. **type=float** конвертирует строку автоматически — все аргументы из терминала приходят как строки.

5. **--help** генерируется автоматически — не нужно писать вручную.

6. **if args.output:** — стандартная проверка. Если не указан → None → False → блок пропускается.

7. **List comprehension** `[int(p.strip()) for p in args.ports.split(',')]` — разбивает строку портов в список чисел за одну строку.

---

## ✅ Чек-лист дня

- [x] `python3 day3.py --help` показывает автогенерированную справку
- [x] Запуск без аргументов выдаёт ошибку про `host`
- [x] Сканирование с `-p 22,80` работает
- [x] Флаг `-v` показывает закрытые порты
- [x] Флаг `-o` сохраняет JSON файл

---

*День 3 из 50 завершён ✅ | Следующий: День 4 — Threading*
