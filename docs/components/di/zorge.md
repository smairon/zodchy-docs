# Zorge
Библиотека для реализации Dependency Injection паттерна. 

## Идея
Довольно часто в коде можно встретить матрешки вида

```python
import typing


class DBConnection:
    def execute(self, query: str):
        pass


class DBEngine:
    def __init__(self, dsn: str):
        self._dsn = dsn

    def connect(self) -> DBConnection:
        pass


class Repository:
    def __init__(self, db_connection: DBConnection):
        self._db_connection = db_connection

    def get_entity(self) -> typing.Any:
        pass


class Service:
    def __init__(self, repository: Repository):
        self._repository = repository

    def do_some_stuff(self):
        pass

    # Теперь в любом месте где мы захотим создать экземпляр сервиса, нам нужно явно инициализировать всю эту пирамиду


def foo_endpoint(dsn: str):
    service = Service(
        repository=Repository(
            db_connection=DBEngine('postgresql://user:pass@127.0.0.1:5432/mydb').connect()
        )
    )


def boo_endpoint():
    service = Service(
        repository=Repository(
            db_connection=DBEngine('postgresql://user:pass@127.0.0.1:5432/mydb').connect()
        )
    )
```
Да, можно внедрять сервис через параметры функции

```python
def foo_endpoint(service: Service):
    pass
```

Но, все равно где-то в коде придется заниматься всей этой утомительной инициализацией
```python
# ... somewhere in code's jungle
service = Service(
    repository=Repository(
        db_connection=DBConnection(
            engine=DBEngine('postgresql://user:pass@127.0.0.1:5432/mydb'),
        )
    )
)
endpoint = foo_endpoint(service)
```

Вот было бы здорово если кто-нибудь мог автоматически резолвить все зависимости 
и возвращать полностью сконструированный объект сервиса. 

## Пример использования
Давайте не будем бездумно бросаться сразу писать реализацию, а начнем с вдумчивого проектирования. 

Предположим, что мы определили в системе парочку бизнес сущностей
```python
# file: entities.py

import datetime
import dataclasses

@dataclasses.dataclass
class Entity:
    id: int


@dataclasses.dataclass
class FooEntity(Entity):
    name: str
    birthdate: datetime.datetime


@dataclasses.dataclass
class BooEntity(Entity):
    title: str
    publish_date: datetime.datetime
```

Заведем файлик contracts.py и там опишем крупными мазками составные 
части нашей архитектуры (в привязке к этим сущностям) и их поведение

```python
# file: contracts.py
import typing
from .entities import (
    Entity, 
    FooEntity, 
    BooEntity
)


class DBConnectionContract(typing.Protocol):
    def execute(self, query: str): ...


class DBEngineContract(typing.Protocol):
    def connect(self) -> DBConnectionContract: ...


class RepositoryContract(typing.Protocol):
    def get_entity(self) -> Entity: ...


class ServiceContract(typing.Protocol):
    def do_some_stuff(self): ...


class FooRepositoryContract(RepositoryContract):
    def get_entity(self) -> FooEntity: ...


class BooRepositoryContract(RepositoryContract):
    def get_entity(self) -> BooEntity: ...
```

Имея такие прекрасные контракты, нам ничто не мешает написать их реализации. Так как python, к счастью, 
включает всю мощь утиной типизации, у нас нет необходимости наследовать эти контракты в классах реализации, 
достаточно воплотить в коду методы, которые есть в протоколах.

Сначала напишем функции, которые нам будут возвращать объекты engine и connection нужных нам контрактов
```python
#file: implementations/storage.py
from ..contracts import DBEngineContract, DBConnectionContract

def create_db_engine(dsn: str) -> DBEngineContract:
    # prepare and return the database engine
    pass

def get_connection(engine: DBEngineContract) -> DBConnectionContract:
    # get connection from engine pool
    pass

def close_connection(connection: DBConnectionContract):
    # do work for graceful shutdown of connection
    pass
```

Далее реализуем репозитории
```python
# file: implementations/repositories.py
from ..entities import FooEntity, BooEntity
class FooRepository:
    def get_entity(self) -> FooEntity:
        # implementation
        pass


class BooRepository:
    def get_entity(self) -> BooEntity:
        # implementation
        pass
```

Видно, здесь модуль с контрактами мы даже не импортируем. Нам достаточно добиться соответствия на уровне поведения - 
реализовать все методы контракта-протокола. Далее реализуем сервис, который работает с данными репозиториями.

```python
# file: implementations/services.py
from ..contracts import FooServiceContract, BooServiceContract

class Service:
    def __init__(self, foo_repo: FooServiceContract, boo_repo: BooServiceContract):
        self._foo_repo = foo_repo
        self._boo_repo = boo_repo
    
    async def do_some_stuff(self):
        # do some stuff with foo and boo repositories 
        pass

```

Важно помнить, что мы по-прежнему импортируем только контракты, хоть реализация этих контрактов и лежит в файле рядом.

И теперь нам нужно где-то связать контракты с их имплементациями. В простейшем случае можно было ло определить 
большой словарик с ключами контрактами и значениями реализациями и передавать его везде в качестве параметра. 
Но, в таком случае вопрос с цепочкой резолвинга пришлось бы каждый раз решать заново 
(копипастой, преимущественно, но все равно не сильно эстетично). 

А что если взять этот словарик, обернуть его классом, назвать контейнером и регистрировать в нем все наши зависимости?
Давайте попробуем

```python
#file: provision.py
import zorge
from . import contracts, implementations

container = zorge.Container()
container.register_dependency(
    contract=contracts.DBEngineContract,
    implementation=implementations.storage.create_db_engine('postgresql://user:pass@127.0.0.1:5432/mydb'),
    cache_scope='container'
)
container.register_dependency(
    contract=contracts.DBConnectionContract,
    implementation=implementations.storage.get_connection,
    cache_scope='resolver'
)
container.register_dependency(
    contract=contracts.FooRepositoryContract,
    implementation=implementations.core.FooRepository
)
container.register_dependency(
    contract=contracts.BooRepositoryContract,
    implementation=implementations.core.BooRepository
)
container.register_dependency(
    contract=contracts.ServiceContract,
    implementation=implementations.core.Service
)
```

И как-бы все, теперь этот контейнер можно передавать в любое место программы и получать 
полностью проинициализированные инстансы по их контрактам.

```python
# file: endpoints.py
import zorge
from . import contracts

async def foo_endpoint(container: zorge.Container):
    async with container.get_resolver() as resolver:
        service = resolver.resolve(contracts.ServiceContract)
        service.do_some_stuff()
```

Резолвер zorge сделан асинхронным, так как синхронные вызовы в асинхронном коде вполне возможны 
(но не забываем про блокировку ивент лупа CPU bound операциями), а вот дискотеку наоборот организовать 
не в пример сложнее. Да и современные сетевые сервисы лучше изначально проектировать асинхронными.