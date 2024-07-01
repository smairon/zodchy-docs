# Types

Набор полезных типов данных

## Empty
Часто нужно указать на отсутствие значения у некоторой сущности. None не подходит, 
так как в некоторых случаях это может быть валидным значением данной сущности. И тут на помощь приходит тип Empty

### Определение

```python
import typing

Empty = typing.NewType('Empty', None)
```

### Использование

```python
import dataclasses
import random

import zodchy


@dataclasses.dataclass
class User:
    id: int
    name: str
    gender: int | None | zodchy.types.Empty = zodchy.types.Empty


users = [
    User(1, "Jhon", 1),
    User(1, "Kate", 2),
    User(1, "Predator", None),
    User(1, "NoBody"),
]

user = users[random.randint(0, 3)]

if user.gender is zodchy.types.Empty:
    print("Please, define your gender")
elif user.gender is None:
    print("What the fuck are you?!")
else:
    print(f"Hello, {"mister" if user.gender == 1 else "madam"} {user.name}")
```