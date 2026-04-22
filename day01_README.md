# День 1 — Сокеты: первый TCP-клиент

**Дата:** 07.04.2026  
**Тема:** Python `socket` модуль — основа любого сетевого инструмента  
**Время:** ~5 часов  

---

## 📖 Теория

### Что такое сокет?

Сокет — это программная "точка подключения" между двумя программами по сети.
Представь сокет как телефонную трубку:

```
1. Берёшь трубку       →  создаёшь сокет
2. Набираешь номер     →  указываешь IP и порт
3. Говоришь            →  отправляешь данные (send)
4. Слушаешь ответ      →  получаешь данные (recv)
5. Кладёшь трубку      →  закрываешь соединение (close)
```

---

### TCP vs UDP

| | TCP | UDP |
|---|---|---|
| Надёжность | Гарантирует доставку | Не гарантирует |
| Скорость | Медленнее | Быстрее |
| Соединение | Устанавливает соединение | Отправляет без соединения |
| Где используется | HTTP, SSH, FTP | DNS, видеозвонки, игры |
| Тип сокета | `SOCK_STREAM` | `SOCK_DGRAM` |

---

### Популярные порты

| Порт | Сервис |
|------|--------|
| 21 | FTP |
| 22 | SSH |
| 25 | SMTP (почта) |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 8080 | HTTP (альт.) |

---

### 3-way Handshake (TCP рукопожатие)

Перед передачей данных TCP делает 3 шага:

```
Клиент                Сервер
  |  ---- SYN ----->  |    "Хочу подключиться"
  |  <-- SYN-ACK ---  |    "Окей, подтверждаю"
  |  ---- ACK ----->  |    "Принял, начинаем"
  |                   |
  |  <-- данные --->  |    Теперь можно общаться
```

Если порт **закрыт** — сервер отвечает `RST` → Python выбрасывает `ConnectionRefusedError`.

---

### Основные методы socket

```python
socket.socket(AF_INET, SOCK_STREAM)  # создать TCP сокет
s.settimeout(3)                       # ждать ответа максимум 3 сек
s.connect((host, port))              # подключиться (3-way handshake)
s.sendall(data)                      # отправить байты
s.recv(1024)                         # получить до 1024 байт
s.close()                            # закрыть соединение
```

> `AF_INET` = Address Family Internet = работа с IPv4 адресами  
> `with socket.socket(...) as s:` — автоматически закрывает сокет

---

### Что такое Banner?

Когда подключаешься к серверу — он часто сам представляется первым:

```
SSH  → SSH-2.0-OpenSSH_8.9p1
FTP  → 220 ProFTPD 1.3.5 Server
HTTP → HTTP/1.1 200 OK
```

Для пентестера это **разведка**: видно версию ПО → ищем уязвимости.

---

### Исключения при работе с сетью

| Исключение | Когда возникает |
|-----------|----------------|
| `ConnectionRefusedError` | Порт закрыт — сервер ответил RST |
| `socket.timeout` | Нет ответа за N секунд (фаервол) |
| `socket.gaierror` | Не удалось определить IP по имени |
| `Exception` | Любая другая ошибка |

---

## 💻 Код — Шаг 1: простой TCP-клиент

Самый базовый скрипт — подключаемся к серверу и читаем ответ.

```python
import socket

HOST = 'scanme.nmap.org'  # легальный сервер nmap для тестов
PORT = 80

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.settimeout(3)
    s.connect((HOST, PORT))
    s.sendall(b'GET / HTTP/1.0\r\nHost: scanme.nmap.org\r\n\r\n')
    data = s.recv(4096)
    print(data.decode('utf-8', errors='ignore'))
```

**Что делает каждая строка:**
- `b'GET / HTTP/1.0\r\n...'` — HTTP запрос в байтах (`b''` = bytes)
- `\r\n` — обязательный перенос строки в протоколе HTTP
- `decode('utf-8', errors='ignore')` — переводим байты в текст

**Результат:**
```
HTTP/1.1 200 OK
Date: Tue, 07 Apr 2026 ...
Server: Apache
...
```

---

## 💻 Код — Шаг 2: многопортовый сканер с banner grabbing

```python
import socket

HOST = '45.33.32.156'  # IP адрес scanme.nmap.org
PORTS = [21, 22, 25, 80, 443, 3306, 8080]

for PORT in PORTS:
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
            s.settimeout(3)
            s.connect((HOST, PORT))

            # HTTP нужно спросить, остальные сами отвечают
            if PORT in [80, 8080]:
                s.sendall(b'GET / HTTP/1.0\r\nHost: scanme.nmap.org\r\n\r\n')

            data = s.recv(1024)
            banner = data.decode('utf-8', errors='ignore').strip()
            print(f'[+] Порт {PORT} ОТКРЫТ | {banner[:60]}')

    except ConnectionRefusedError:
        print(f'[-] Порт {PORT} закрыт')
    except socket.timeout:
        print(f'[?] Порт {PORT} таймаут (фильтруется фаерволом)')
    except Exception as e:
        print(f'[!] Порт {PORT} ошибка: {e}')
```

**Результат:**
```
[?] Порт 21  таймаут (фильтруется фаерволом)
[+] Порт 22  ОТКРЫТ | SSH-2.0-OpenSSH_6.6.1p1 Ubuntu-2ubuntu2.13
[?] Порт 25  таймаут (фильтруется фаерволом)
[+] Порт 80  ОТКРЫТ | HTTP/1.1 200 OK
...
```

---

## 💻 Код — Финальный: профессиональный сканер

Финальная версия с выравниванием, счётчиками и красивым выводом.

```python
import socket

# --- конфигурация ---
HOST = '45.33.32.156'
PORTS = [21, 22, 23, 25, 80, 110, 443, 3306, 5432, 8080]

# --- счётчики ---
open_count = 0
closed_count = 0
filtered_count = 0

# --- сканирование ---
for PORT in PORTS:
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
            s.settimeout(3)
            s.connect((HOST, PORT))

            if PORT in [80, 8080]:
                s.sendall(b'GET / HTTP/1.0\r\nHost: scanme.nmap.org\r\n\r\n')

            data = s.recv(1024)
            banner = data.decode('utf-8', errors='ignore').strip()
            print(f'[+] {PORT:<5} OPEN   | {banner[:50]}')
            open_count += 1

    except ConnectionRefusedError:
        print(f'[-] {PORT:<5} CLOSED |')
        closed_count += 1
    except socket.timeout:
        print(f'[?] {PORT:<5} FILTER |')
        filtered_count += 1
    except Exception as e:
        print(f'[!] {PORT:<5} ERROR  | {e}')

# --- итог ---
print(f'\n{"=" * 35}')
print(f'  Сканирование завершено')
print(f'  Открытых:    {open_count}')
print(f'  Закрытых:    {closed_count}')
print(f'  Фильтруется: {filtered_count}')
print(f'{"=" * 35}')
```

**Результат:**
```
[?] 21    FILTER |
[+] 22    OPEN   | SSH-2.0-OpenSSH_6.6.1p1 Ubuntu-2ubuntu2.13
[?] 23    FILTER |
[?] 25    FILTER |
[+] 80    OPEN   | HTTP/1.1 200 OK
[?] 110   FILTER |
[?] 443   FILTER |
[?] 3306  FILTER |
[?] 5432  FILTER |
[?] 8080  FILTER |

===================================
  Сканирование завершено
  Открытых:    2
  Закрытых:    0
  Фильтруется: 8
===================================
```

---

## 🔑 Ключевые выводы дня

1. **Сокет** — это просто программный интерфейс для сетевого соединения.

2. **`settimeout()`** — обязателен в любом сканере. Без него скрипт зависает навсегда на закрытых портах.

3. **Banner grabbing** — бесплатная разведка. Сервер сам говорит свою версию при подключении.

4. **`try/except`** — без обработки ошибок первый же закрытый порт убьёт скрипт.

5. **`FILTER`** означает фаервол — порт может быть открыт, но фаервол блокирует наши пакеты.

6. **`{PORT:<5}`** в f-строке — выравнивание по левому краю в 5 символов. Делает вывод аккуратным.

---

## ✅ Чек-лист дня

- [x] Понял что такое сокет и как он работает
- [x] Знаю разницу между TCP и UDP
- [x] Понимаю 3-way handshake
- [x] Написал TCP-клиент который подключается к серверу
- [x] Реализовал banner grabbing для нескольких портов
- [x] Добавил обработку всех типов сетевых ошибок
- [x] Сделал красивый вывод с выравниванием и счётчиками
- [x] Проверил результаты на scanme.nmap.org

---

## 📚 Ресурсы

- [Python socket документация](https://docs.python.org/3/library/socket.html)
- [scanme.nmap.org](http://scanme.nmap.org) — легальный сервер для тестов
- Книга: **Black Hat Python** — Justin Seitz (глава 2)
- [TryHackMe: Scripting room](https://tryhackme.com)

---

*День 1 из 50 завершён ✅*
