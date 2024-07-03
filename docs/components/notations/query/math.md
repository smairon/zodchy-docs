# Query math notation

Нотация описания запроса к данным, вдохновленная математической пунктуацией. 
Данные записываются в одну строку в виде 
```text
k1=v1&k2=v2..&kn=vn
```
- **ki** - название i-ого параметра
- **vi** - токен i-ого значения

В состав токена помимо собственно значения входят символы, определяющие операцию над этим значением

## Операции
### Равенство
```text
user_id=1
created_at=2024-07-04T12:12:12
birth_date=1970-01-01
```
Задает равенство значения параметра операнду справа от операторв **=**

### Диапазон
```text
birth_date=[1970-01-01,2000-01-01] # same as: 1970-01-01 <= birth_date <= 2000-01-01 
birth_date=(1970-01-01,2000-01-01] # same as: 1970-01-01 < birth_date <= 2000-01-01 
birth_date=[1970-01-01,2000-01-01) # same as: 1970-01-01 <= birth_date < 2000-01-01 
birth_date=(1970-01-01,2000-01-01) # same as: 1970-01-01 < birth_date < 2000-01-01 
```
Задает диапазон который может принимать значение параметра. 
По аналогии с математическими открытым, полуоткрытым интервалами и отрезком

### Больше/Меньше

```text
birth_date=[1970-01-01,) # same as: birth_date >= 1970-01-01
birth_date=(1970-01-01,) # same as: birth_date > 1970-01-01
birth_date=(,2000-01-01] # same as: birth_date <= 2000-01-01
birth_date=(,2000-01-01) # same as: birth_date < 2000-01-01
```
Операции неравенства задаются через указание диапазона, в котором одна из границ отсутствует 
(т.е. бесконечна с соотвествующим знаком )

### Множество

```text
tag_id={1,2,3,4} # same as: tag_id in (1,2,3,4)
user_id={1,2} # same as: user_id in (1, 2)    
```
Задает ограниченное множество вариантов значения параметра

### Поиск по шаблону

```text
name=~james # same as: name ilike '%james%'
name=~~james # same as: name like '%james%'
```
Задает шаблон поиска по значению параметра

### Отрицание (инвертирование)
```text
birth_date=!1970-01-01 # same as: birth_date != 1970-01-01
tag_id=!{1,2,3,4} # same as: tag_in not in (1,2,3,4)
name=!~james # same as: name not ilike '%james%'
birth_date=![1970-01-01, 2000-01-01] # same as: birth_date < 1970-01-01 or birth_date > 2000-01-01
```
Задает инвертирование последующей операции над значением параметра

## Парсер
Для стека zodchy разработан 
[парсер](https://github.com/smairon/zodchy-notations/blob/main/zodchy_notations/query/math.py) данной нотации

### Пример использования

```python
import datetime

import zodchy_notations

parser = zodchy_notations.query.math.Parser()

query = "user_id=1&state={active,pending}&created_at=(,2024-07-01T00:00:00)"
tokens = list(
    parser(
        query=query, 
        types_map={'user_id': int, 'state': str, 'created_at': datetime.datetime}
    )
)
# Result:
# [
#   (user_id, zodchy.operators.EQ(1)),
#   (state, zodchy.operators.SET('active', 'pending')),
#   (
#       created_at, 
#       zodchy.operators.RANGE(
#           None, 
#           zodchy.operators.LT(datetime.datetime(2024, 07, 1))
#       )
#    )
# ]
```
Далее набор этих кортежей можно скормить на вход некоторого адаптера, который преобразует 
последовательность этих токенов в запрос на любом другом языке (sql, mongoQL, Redis API, plain text filters, etc)

В качестве примера можно рассмотреть 
[адаптер](https://github.com/smairon/zodchy-fastapi/blob/main/zodchy_fastapi/request/adapter.py) 
Request моделей FastAPI в [сущности](../../../base/codex.md) CQEA архитектуры.