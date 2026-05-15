# День 12 — Сетевой сниффер с Scapy 📡

## Что изучили
- Перехват сетевых пакетов через `scapy`
- Анализ IP заголовков (src, dst, proto)
- Фильтрация трафика по протоколу
- Сохранение пакетов в JSON

---

## Итоговый код — day12.py

```python
from scapy.all import sniff, IP, TCP, UDP
import argparse, json

parser = argparse.ArgumentParser()
parser.add_argument("--count", type=int, default=10)
parser.add_argument("--filter", default='tcp')
args = parser.parse_args()
packets = []

def process(packet):
    if packet.haslayer(IP):
        src = packet[IP].src
        dst = packet[IP].dst
        proto = packet[IP].proto
        with open("packets.json", "a") as f:
            json.dump({"src": src, "dst": dst, "proto": proto}, f)
            f.write("\n")
        print(f"[+] {src} → {dst} | proto: {proto}")

print(f"[*] Слушаем {args.filter} трафик... ({args.count} пакетов)")
sniff(filter=args.filter, prn=process, count=args.count)
```

---

## Как запустить

```bash
# Базовый запуск (нужен sudo!)
sudo python3 day12.py

# С параметрами
sudo python3 day12.py --count 5 --filter tcp
sudo python3 day12.py --count 20 --filter udp
sudo python3 day12.py --count 10 --filter icmp

# Создать трафик для теста
curl http://google.com

# Посмотреть результат
cat packets.json
```

---

## Пример вывода

```
[*] Слушаем tcp трафик... (10 пакетов)
[+] 10.0.2.15 → 51.83.41.68 | proto: 6
[+] 51.83.41.68 → 10.0.2.15 | proto: 6
[+] 10.0.2.15 → 51.83.41.68 | proto: 6
[+] 51.83.41.68 → 10.0.2.15 | proto: 6
```

---

## Ключевые концепции

### sniff() — главная функция
```python
sniff(
    filter="tcp",    # какой трафик ловить
    prn=process,     # функция для каждого пакета
    count=10         # сколько пакетов поймать
)
```

### Слои пакета
```python
packet.haslayer(IP)   # есть ли IP слой?
packet[IP].src        # IP источника
packet[IP].dst        # IP назначения
packet[IP].proto      # номер протокола
```

### Номера протоколов
| Номер | Протокол |
|---|---|
| `6` | TCP |
| `17` | UDP |
| `1` | ICMP |

### Фильтры Scapy
```bash
"tcp"   # только TCP пакеты
"udp"   # только UDP пакеты
"icmp"  # только ICMP (ping)
"port 80"  # только порт 80
```

### Почему нужен sudo?
```
Перехват пакетов = доступ к сетевому интерфейсу
Сетевой интерфейс = системный ресурс
Системный ресурс = только root!
```

---

## Что видно в трафике

```
10.0.2.15   → 51.83.41.68  ← твой Kali отправляет запрос
51.83.41.68 → 10.0.2.15   ← сервер отвечает
```
Это TCP handshake — пакеты идут туда и обратно!

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
| День 12 | Сетевой сниффер с Scapy | ✅ |

---

*Python для Offensive Security — 50 дней*
