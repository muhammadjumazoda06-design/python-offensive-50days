# 📦 День 4 — Threading с ThreadPoolExecutor

## 🎯 Что изучил сегодня

- Что такое **поток (Thread)** и зачем он нужен
- Как использовать **ThreadPoolExecutor** для параллельного сканирования
- Зачем нужен **Lock** и как он защищает данные
- Как выносить логику в **отдельную функцию** (`def`)

---

## ⚡ Проблема без Threading

```
порт 22  → ждём 3 сек
порт 80  → ждём 3 сек  
порт 443 → ждём 3 сек
= 9 секунд для 3 портов
```

100 портов × 3 сек = **5 минут** 😴

---

## 🚀 Решение — Threading

```
порт 22  ┐
порт 80  │ все одновременно = ~3 секунды
порт 443 ┘
```

---

## 🔑 Ключевые концепции

### def — функция
```python
def scan_port(port):
    # логика сканирования одного порта
```
Выносим логику в функцию чтобы ThreadPoolExecutor мог вызывать её параллельно.

### ThreadPoolExecutor
```python
with ThreadPoolExecutor(max_workers=100) as executor:
    executor.map(scan_port, args.ports)
```
- `max_workers=100` — максимум 100 потоков одновременно
- `executor.map()` — запускает функцию для каждого порта параллельно
- `with` — автоматически закрывает все потоки когда закончит

### Lock — защита от конфликтов
```python
lock = threading.Lock()

with lock:
    print(...)        # только один поток печатает за раз
    results.append()  # только один поток пишет за раз
```
Без Lock потоки перебивают друг друга при записи.

---

## 📝 Полный скрипт

```python
import socket, time, json, argparse, threading
from concurrent.futures import ThreadPoolExecutor

parser = argparse.ArgumentParser(description='Port Scanner')
parser.add_argument('host')
parser.add_argument('--timeout', type=int, default=3)
parser.add_argument('--ports', nargs='+', type=int, default=[22, 80, 443])
args = parser.parse_args()

results = []
lock = threading.Lock()

def scan_port(port):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.settimeout(args.timeout)
    try:
        s.connect((args.host, port))
        time.sleep(0.5)
        data = s.recv(1024)
        banner = data[:50].decode('utf-8', errors='ignore')
        with lock:
            print(f'[+] {port} OPEN | {banner}')
            results.append({'port': port, 'status': 'OPEN', 'banner': banner})
    except ConnectionRefusedError:
        with lock:
            print(f'[-] {port} CLOSED')
            results.append({'port': port, 'status': 'CLOSED', 'banner': ''})
    except socket.timeout:
        with lock:
            print(f'[?] {port} FILTERED')
            results.append({'port': port, 'status': 'FILTERED', 'banner': ''})
    finally:
        s.close()

with ThreadPoolExecutor(max_workers=100) as executor:
    executor.map(scan_port, args.ports)

with open("report.json", "w") as f:
    json.dump(results, f, indent=4)

print("Отчёт сохранён в report.json")
```

---

## 🖥️ Использование

```bash
# Базовое сканирование
python3 day4.py 45.33.32.156

# С указанием портов
python3 day4.py 45.33.32.156 --ports 22 80 443 8080 3306

# С кастомным таймаутом
python3 day4.py 45.33.32.156 --ports 22 80 443 --timeout 1
```

---

## 📊 Пример вывода

```
[+] 22 OPEN | SSH-2.0-OpenSSH_6.6.1p1 Ubuntu-2ubuntu2.13
[?] 443 FILTERED
[?] 8080 FILTERED
[?] 3306 FILTERED
[?] 21 FILTERED
[?] 25 FILTERED
[?] 80 FILTERED
Отчёт сохранён в report.json
```

> ⚠️ Порядок вывода случайный — это нормально! Потоки завершаются не по порядку.

---

## 📈 Прогресс курса

| День | Тема | Статус |
|------|------|--------|
| День 1 | Сокеты, TCP, Banner Grabbing | ✅ |
| День 2 | JSON отчёты | ✅ |
| День 3 | argparse CLI | ✅ |
| День 4 | Threading | ✅ |
