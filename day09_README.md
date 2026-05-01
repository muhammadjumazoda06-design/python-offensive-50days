# День 9 — DNS Разведка 🌐

## Что изучили
- DNS записи разных типов (A, MX, NS, TXT)
- Библиотека `dnspython`
- Автоматический перебор типов записей
- Сохранение результатов в JSON

---

## Итоговый код — day9.py

```python
import dns.resolver, argparse, json

parser = argparse.ArgumentParser()
parser.add_argument('domain')
args = parser.parse_args()

record_types = ['A', 'MX', 'NS', 'TXT']
results = {}

for rtype in record_types:
    try:
        records = dns.resolver.resolve(args.domain, rtype)
        results[rtype] = []
        for rdata in records:
            results[rtype].append(str(rdata))
            print(f"[+] {rtype}: {rdata}")
    except:
        print(f"[-] {rtype}: не найдено")

with open(f"{args.domain}_dns.json", 'w') as f:
    json.dump(results, f, indent=4)
```

---

## Как запустить

```bash
# Установить библиотеку
pip install dnspython --break-system-packages

# Запустить сканер
python3 day9.py google.com

# Результат сохранится в файл
cat google.com_dns.json
```

---

## Пример вывода

```
[+] A: 142.250.178.110
[+] MX: 10 smtp.google.com.
[+] NS: ns1.google.com.
[+] NS: ns2.google.com.
[+] NS: ns3.google.com.
[+] NS: ns4.google.com.
[+] TXT: v=spf1 include:_spf.google.com ~all
[+] TXT: facebook-domain-verification=...
[+] TXT: apple-domain-verification=...
[-] ...
```

---

## Типы DNS записей

| Тип | Что содержит | Ценность для пентестера |
|---|---|---|
| `A` | IP адрес сервера | Прямой адрес цели |
| `MX` | Почтовый сервер | Атаки на почту |
| `NS` | DNS серверы домена | Инфраструктура |
| `TXT` | Текстовые данные | Все используемые сервисы |

---

## Что раскрывают TXT записи

```
v=spf1 include:_spf.google.com     ← защита от подделки email
google-site-verification=...        ← верификация в Google
facebook-domain-verification=...    ← использует Facebook
apple-domain-verification=...       ← использует Apple сервисы
MS=E4A68B...                        ← использует Microsoft сервисы
docusign=...                        ← использует DocuSign
```

**Вывод:** TXT записи раскрывают все сервисы компании — без единого запроса к серверу! 🎯

---

## Новые команды

```python
# Импорт библиотеки
import dns.resolver

# DNS запрос
records = dns.resolver.resolve(args.domain, 'A')

# Обход результатов
for rdata in records:
    print(str(rdata))

# Словарь для хранения по типу
results = {}
results['A'] = []
results['A'].append(str(rdata))
```

---

## Почему свои NS серверы безопаснее?

| | Чужие NS | Свои NS |
|---|---|---|
| Контроль | Частичный | Полный |
| Безопасность | Зависит от провайдера | Сам настраиваешь |
| Пример | ns1.reg.ru | ns1.google.com |

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

---

*Python для Offensive Security — 50 дней*
