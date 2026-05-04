# День 10 — WHOIS Разведка 🕵️

## Что изучили
- WHOIS запросы через библиотеку `python-whois`
- Получение данных о владельце домена
- Работа с полями объекта whois
- Сохранение результатов в JSON

---

## Итоговый код — day10.py

```python
import whois, argparse, json

parser = argparse.ArgumentParser()
parser.add_argument('domain')
args = parser.parse_args()

w = whois.whois(args.domain)

print(f"\n[*] WHOIS: {args.domain}")
print(f"  Registrar:    {w.registrar}")
print(f"  Created:      {w.creation_date}")
print(f"  Expires:      {w.expiration_date}")
print(f"  Country:      {w.country}")
print(f"  Emails:       {w.emails}")

results = {
    "domain": args.domain,
    "registrar": w.registrar,
    "country": w.country,
    "emails": w.emails
}
with open(f"{args.domain}_whois.json", 'w') as f:
    json.dump(results, f, indent=4)
```

---

## Как запустить

```bash
# Установить библиотеку
pip install python-whois --break-system-packages

# Запустить
python3 day10.py google.com

# Результат сохранится в файл
cat google.com_whois.json
```

---

## Пример вывода

```
[*] WHOIS: google.com
  Registrar:    MarkMonitor, Inc.
  Created:      1997-09-15
  Expires:      2028-09-14
  Country:      US
  Emails:       ['abusecomplains@markmonitor.com']
```

---

## Поля WHOIS объекта

| Поле | Что содержит |
|---|---|
| `w.domain_name` | Имя домена |
| `w.registrar` | Кто зарегистрировал домен |
| `w.creation_date` | Дата создания домена |
| `w.expiration_date` | Дата истечения регистрации |
| `w.country` | Страна владельца |
| `w.emails` | Email контакты |

---

## Что делает пентестер с данными WHOIS

| Данные | Применение |
|---|---|
| `emails` | Фишинг, OSINT, брутфорс почты |
| `registrar` | Определение инфраструктуры |
| `creation_date` | Старый домен = больше доверия |
| `country` | Юрисдикция, законы |

---

## Ключевые концепции

### whois.whois()
```python
w = whois.whois("google.com")
# w — объект со всеми данными домена
print(w.registrar)      # MarkMonitor, Inc.
print(w.country)        # US
print(w.emails)         # ['abuse@markmonitor.com']
```

### Почему некоторые домены возвращают None?
```
asu.tut.tj → None  # домен .tj не поддерживает публичный WHOIS
```
Некоторые страны и регистраторы скрывают данные — это называется **WHOIS Privacy**.

### Почему даты выглядят странно?
```python
Created: [datetime.datetime(1997, 9, 15, 4, 0), ...]
```
Иногда WHOIS возвращает **список дат** — разные серверы дают разные форматы. Берём первый элемент: `w.creation_date[0]`

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

---

*Python для Offensive Security — 50 дней*
