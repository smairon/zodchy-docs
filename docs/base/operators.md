# Operators
[Набор классов](https://github.com/smairon/zodchy/blob/main/zodchy/operators.py) для декларации операций с данными.

Вместо тысячи абстрактных слов ниже приведен пример кода, для изучения операторов в реальном контексте. 
Определен класс параметра, который на деле может быть чем угодно (полем БД, элементом  query в http запросе  и тд). 
У параметра есть имя и есть operator который будет применен к данным этого параметра. 
В комментариях к каждой строке указан математический смысл оператора

```python
import datetime
import dataclasses

from zodchy import operators
from zodchy import codex


@dataclasses.dataclass
class Parameter:
    name: str
    operator: codex.query.ClauseBit


@dataclasses.dataclass
class FilterIntParameter(Parameter):
    operator: codex.query.FilterBit[int]


@dataclasses.dataclass
class FilterDateParameter(Parameter):
    operator: codex.query.FilterBit[datetime.datetime]


@dataclasses.dataclass
class FilterStrParameter(Parameter):
    operator: codex.query.FilterBit[str]


@dataclasses.dataclass
class OrderParameter(Parameter):
    operator: codex.query.OrderBit


@dataclasses.dataclass
class SliceParameter(Parameter):
    operator: codex.query.SliceBit


param1 = FilterIntParameter("user_id", operators.EQ(1))  # user_id == 1
param2 = FilterIntParameter("user_id", operators.NE(1))  # user_id != 1
param3 = FilterIntParameter("user_id", operators.LE(1))  # user_id <= 1
param4 = FilterIntParameter("user_id", operators.GE(1))  # user_id >= 1
param5 = FilterIntParameter("user_id", operators.LT(1))  # user_id < 1
param6 = FilterIntParameter("user_id", operators.GT(1))  # user_id > 1
param7 = FilterIntParameter("user_id", operators.IS(None))  # user_id is None

param8 = FilterIntParameter("user_id", operators.SET(1, 2, 3))  # user_id in (1, 2, 3)

param9 = FilterDateParameter(
    "birth_date",
    operators.RANGE(
        operators.GE(datetime.datetime(1980, 1, 1)),
        operators.LT(datetime.datetime(1990, 12, 31))
    )
)  # 1980-01-01 >= birth_date <= 1990-12-31

param10 = FilterStrParameter("name", operators.LIKE('jhon', case_sensitive=True))  # user_id ilike 'jhon'
param11 = FilterStrParameter("name", operators.LIKE('jhon', case_sensitive=False))  # user_id like 'jhon'

param12 = FilterIntParameter("status_id", operators.NOT(operators.SET(1, 2)))  # status_id not in (1, 2)

param13 = OrderParameter(
    "created_at", 
    operators.DESC(1)
)  # order dataset by created_at with priority 1 (the smaller the priority)

param14 = SliceParameter("limit", operators.LIMIT(100))  # limit dataset with 100 rows
param15 = SliceParameter("offset", operators.OFFSET(10))  # skip 10 rows of dataset
```
Используя заданные выше параметры, можно сконструировать, к примеру, простой декларативный запрос к данным
```python
query = (
    param4, 
    param9,
    param12,
    param13,
    param14
)
bucket = "users"
result = some_kind_of_client.get_data(bucket, query)
```
Здесь мы просим некоторой клиент сходить в некий бакет хранения "users" и вернуть 100 пользователей с id >= 4, 
родившихся в 80-ые годы прошлого века, со статусом не 1 и не 2, отсортированных по дате создания.

Заметим, что нас абсолютно не интересут тип хранилища данных. Это может быть что угодно и где угодно, 
конкретный протокол доступа к данным определяется целиком внутри some_kind_of_client и все что нужно - преобразовать 
iterable  of params в семантику запроса к конкретному источнику (sql, mongo json, redis API и тд)

Здесь мы намеренно не касаемся указания полей в итоговом датасете, так как к семантике операторов это не имеет отношения.