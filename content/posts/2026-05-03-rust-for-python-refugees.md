---
title: 'Rust для беженцев с Python - 3'
author: ["Stanislav"]
date: 2026-05-03T11:00:00+00:00
url: /2026/05/rust-for-python-refugees-3/
categories:
  - Tech
tags:
  - rust
  - python
---

Едем дальше? Ведь на защите от переполнения u8 и match далеко не уехать. Теперь пора заняться тем, без чего не живет приложенька: структурами данных. В Python говорим данные, подразумеваем словари (`Dict`), списки (`List`) и классы с данными `dataclasses`. В Rust в структурах, перечислениях и векторах. Разница не только в названиях.

## Списки, кортежи, массивы, два вектора

В Python список `list` инициализируется `[]` и представляет из себя швейцарский нож. И очередь, и стек, и массив, и буфер. По своей сути, это гетерогенный динамический массив указателей. Поскольку список в Python хранит не сами объекты, а ссылки на них, туда можно наваливать что угодно. И строки, и числа, и другие списки. 

```python
users = [['Stas',38,['systems-architect','systems-engineer']]]
```

Стек работает по принципу LIFO (Last-in, First-out), очередь FIFO (First-in, First-out). 

```python
users.append(['Danil', 38, ['team-lead']])
```

Благодаря резервированию памяти список отлично подходит для временного хранилища, но есть и подводные. 

Если использовать список как стек, то `pop()` и `append()` с конца выполняются за `O(1)`. А вот если как очередь с `pop(0)` и `append(x)` то начинается боль с `O(n)`, так как заставляет интерпретатор сдвигать все оставшиеся элементы на одну позицию. Резервирование памяти на вырост так же больно бьет по потреблению RAM, значительно подорожавшей из-за нейрослопа. 

И вот тут доверчивый Python-разработчик наступает в ржавую бюрократию, где намерения надо четко фиксировать конкретной структурой данных:

* массив если размер известен заранее
* кортеж если нужно быстро сгруппировать несколько значений
* Vec<T> если нужен обычный растущий список
* VecDeque<T> если нужна очередь с быстрым добавлением и удалением с двух сторон

### Массивы

Массив в Rust это набор элементов одного типа и фиксированной длины. Размер массива является частью типа, поэтому массив из двух чисел и массив из четырёх чисел в Rust две разные сущности. И это не попытка испортить настроение программисту, это структура для случаев, когда рост и не нужен. Такие дела. 

```rust
fn main() {
    let ports: [u16; 4] = [80, 443, 8080, 8443];
    println!("{:?}", ports);
}
```

Если элементы массива реализуют trait `Copy` то и массив просто копируется

```rust
fn main() {
    let ports: [u16; 4] = [80, 443, 8080, 8443];
    let backup = ports;  // скопирован

    println!("original: {:?}", ports);
    println!("backup:   {:?}", backup);
}
```

А если `Clone`, то надо убедиться, что массив склонирован:

```rust
fn main() {
    let services: [String; 2] = [
        String::from("nova-api"),
        String::from("neutron-api"),
    ];
    let backup = services.clone();   // склонирован
    
    println!("{:?}", services);
    println!("{:?}", backup);
}
```

Индексация за границами массива ловится через `get()` и `Option`:

```rust
fn main() {
    let ports: [u16; 4] = [80, 443, 8080, 8443];

    // println!("{}", ports[10]);  // паника

    match ports.get(10) {
        Some(p) => println!("port: {}", p),
        None => println!("нет такого индекса"),
    }
}
```

### Кортежи 

В Python кортеж неизменяемый:

```python
user = ("Stas", 38, ["systems-engineer", "systems-architect"])
user[0] = "Danil"  # TypeError
user[2].append("shitposter")   # а вот так можно
```

В Rust кортеж тоже собирает несколько значений в одну структуру, но акцент немного другой. Это не просто "список, который нельзя менять", а быстрый способ сгруппировать значения разных типов без объявления отдельной структуры.

```rust
fn main() {
    let user: (&str, u8, [&str; 2]) = (
        "Stas",
        38,
        ["systems-engineer", "systems-architect"],
    );

    println!("name:  {}", user.0);
    println!("age:   {}", user.1);
    println!("roles: {:?}", user.2); // Debug вместо Display для массива
    println!("roles: {}", user.2.join(", ")); // или через join, если надо по красоте 
}
```

И тут начинается главный минус кортежей. Пока полей два или три, то жить можно. Но когда в коде появляется user.0, user.1, user.n, через неделю приходится вспоминать, кто все эти люди.

Распаковка схожая:

```python
user = ("Stas", 38, ["engineer", "architect"])
name, age, roles = user
```

```rust
fn main() {
    let user: (&str, u8, [&str; 2]) = (
        "Stas",
        38,
        ["systems-engineer", "systems-architect"],
    );
    let (name, age, roles) = user;

    println!("{} is {} years old, has roles={:?}", name, age, roles);
}
```

Если часть значений не нужна, то можно использовать `_`:

```rust
let (name, _, roles) = user;
```

Или можно отбросить хвост:

```rust
let (name, ..) = user;
```

### Вектор `Vec<T>`

Если после Python хочется обычный растущий список, то в Rust чаще всего нужен `Vec<T>`. Это динамически изменяемый массив однородных данных, который инициализируется либо через `Vec::new()`:

```rust
fn main() {
    let mut ports: Vec<u16> = Vec::new();

    ports.push(80);
    ports.push(443);
    ports.push(8080);

    println!("{:?}", ports);
}
```

или через макрос `vec!`:

```rust
fn main() {
    let ports = vec![80, 443, 8080, 8443];

    println!("{:?}", ports);
}
```

Буква `T` в `Vec<T>` означает “какой-то конкретный тип”. Например, `Vec<u16>`, `Vec<String>`, `Vec<User>`. Вектор чисел, вектор строк, вектор пользователей. Но не солянка из строки, числа и списка ролей.

Вектор характеризуется длиной (сколько элементов сейчас лежит внутри) и емкостью (сколько элементов можно положить без нового выделения памяти):

```rust
fn main() {
    let mut ports = Vec::new();

    println!("len={}, capacity={}", ports.len(), ports.capacity());

    ports.push(80);
    ports.push(443);

    println!("len={}, capacity={}", ports.len(), ports.capacity());
}
```

Когда емкости перестаёт хватать, вектор выделяет новый кусок памяти жирнее и переносит туда данные. Поэтому добавление в конец обычно дешёвое, но иногда надо переехать. Соскучились по Borrow Checker?

```rust
fn main() {
    let mut ports = vec![80, 443, 8080];

    let first = &ports[0];

    ports.push(8443); // Ошибка: mutable borrow

    println!("{}", first); 
}
```

Именно из-за этого Rust осторожен с ссылками на элементы вектора во время изменения вектора. Если во время `push()` данные переедут, старая ссылка будет указывать туда, где уже ничего нет. Поэтому правило простое: либо сначала читать, потом модифицировать, либо копировать/клонировать значение, если оно нужно после изменения вектора.

С индексацией всё привычно:

```rust
fn main() {
    let ports = vec![80, 443, 8080];

    println!("{}", ports[0]);
}
```

Но если обратиться за пределы вектора, случится паника. Безопасный вариант, как обычно, `get()`, который возвращает `Option`. 

### Вектор `VecDeque<T>` 

Если `Vec<T>` это обычный растущий список, то `VecDeque<T>` это очередь с двумя концами. Для питониста ближайшая аналогия `collections.deque`. Технически `VecDeque<T>` устроен как кольцевой буфер. Это значит, что начало и конец вектора могут "ездить" внутри заранее выделенной памяти, а элементы не нужно постоянно сдвигать при добавлении или удалении спереди.

```python
queue = ["task-1", "task-2", "task-3"]
queue.pop(0)        # "task-1", но O(n) сдвиг всего хвоста
queue.insert(0, "urgent")  # опять O(n)
```

В Rust:

```
Начальное состояние, при capacity(8):
[ task-1 | task-2 | task-3 | _ | _ | _ | _ | _ ]
  ^ head                     ^ tail

После pop_front():
[ _ | task-2 | task-3 | _ | _ | _ | _ | _ ]
      ^ head            ^ tail

После push_back("task-4"):
[ _ | task-2 | task-3 | task-4 | _ | _ | _ | _ ]
      ^ head                     ^ tail
```

```rust
use std::collections::VecDeque;

fn main() {
    let mut queue = VecDeque::with_capacity(8);

    queue.push_back("task-1");
    queue.push_back("task-2");

    queue.push_front("urgent-task");

    println!("{:?}", queue);

    println!("next:      {:?}", queue.pop_front());
    println!("remaining: {:?}", queue);
    println!("last:      {:?}", queue.pop_back());
}
```

Если продолжать push_front/pop_back с энтузиазмом, то рано или поздно хвост дойдет до конца буфера и вернется в начало:

```
[ task-5 | _ | _ | task-1 | task-2 | task-3 | _ | _ ]
           ^ tail  ^ head
```           

`VecDeque` может хранить данные не одним непрерывным куском, а двумя срезами. Логически это одна очередь, но физически хвост может вернуться в начало буфера. Это можно увидеть через as_slices():

```rust
use std::collections::VecDeque;

fn main() {
    let mut queue = VecDeque::with_capacity(5);

    queue.push_back("task-1");
    queue.push_back("task-2");
    queue.push_back("task-3");
    queue.push_back("task-4");
    queue.push_back("task-5");

    queue.pop_front();
    queue.pop_front();

    queue.push_back("task-6");
    queue.push_back("task-7");

    println!("logical: {:?}", queue);

    let (left, right) = queue.as_slices();

    println!("first slice:  {:?}", left);
    println!("second slice: {:?}", right);
}
```

```
[ task-6 | task-7 | task-3 | task-4 | task-5 ]
  ^^^^^^   ^^^^^^   ^^^^^^   ^^^^^^   ^^^^^^
  second   second    first    first    first
```

Поэтому при `pop_front()` ему не нужно сдвигать все элементы влево, достаточно передвинуть указатель начала. Это важно помнить: VecDeque быстрый для очередей, но если нужен случайный доступ по индексу или непрерывный срез всего содержимого, то Vec тут будет чуть быстрее по скорости.

| Задача                      | Структура                                   |
| --------------------------- | ------------------------------------------- |
| Стек (LIFO)                 | `Vec`: `push`/`pop` с конца                 |
| Очередь (FIFO)              | `VecDeque`: `push_back`/`pop_front`         |
| Случайный доступ по индексу | `Vec`, так как `VecDeque` медленнее         |


## Словари и хешмапы

В Rust `HashMap<K, V>` эквивалент `dict` в Python. 

```python
services = {}

services["nova-api"] = 8774
services["neutron-api"] = 9696
services["keystone"] = 5000
```

Но с приправой Rust. Не "ключи +/- строки, а значения что бог пошлет", а конкретно: `HashMap<String, u16>`:

```rust
use std::collections::HashMap;

fn main() {
    let mut services: HashMap<String, u16> = HashMap::new();

    services.insert(String::from("nova-api"), 8774);
    services.insert(String::from("neutron-api"), 9696);
    services.insert(String::from("keystone"), 5000);

    println!("{:?}", services);
}
```

или `HashMap<String, Vec<String>>`:

```rust
use std::collections::HashMap;

fn main() {
    let mut services: HashMap<String, Vec<String>> = HashMap::new();

    services.insert(
        String::from("nova-api"),
        vec![String::from("public"), String::from("internal"), String::from("admin")],
    );

    services.insert(
        String::from("keystone"),
        vec![
            String::from("public"),
            String::from("internal"),
        ],
    );
}
```

или `HashMap<&str, &str>`:

```rust
use std::collections::HashMap;

fn main() {
    let mut services: HashMap<&str, &str> = HashMap::new();

    services.insert("nova-api", "compute");
    services.insert("neutron-api", "network");
    services.insert("keystone", "identity");

    println!("{:?}", services);
}
```

Если запутались в чем разница `&str` и `String`, то имеет смысл открыть снова первую статью.

Доступ к данным осуществляется похожим образом с Python, где services["missing"] вызовет KeyError, а services.get("missing") отдаст None.

```rust
use std::collections::HashMap;

fn main() {
    let mut services: HashMap<&str, u16> = HashMap::new();
    services.insert("nova-api", 8774);

    match services.get("nova-api") {
        Some(port) => println!("порт: {}", port),
        None => println!("сервис не найден"),
    }

    // services["missing"]; // паника, как с KeyError
}
```

Если нужно значение по умолчанию, то использует известный по второй статье `unwrap_or()`:

```rust
let port = services.get("cinder-api").unwrap_or(&8774);
```

Поведение при добавление тоже, в общем, логичное. В Python переменная `name` осталась бы жить, в Rust она переезжает в HashMap:

```rust
use std::collections::HashMap;

fn main() {
    let mut services = HashMap::new();
    let name = String::from("nova-api");

    services.insert(name, 8774);

    // println!("{}", name); // ошибка: value moved
    println!("{:?}", services);
}
```

Если добавляем списки, то, пожалуй, даже проще:

```python
services = {}

if "nova-api" not in services:
    services["nova-api"] = []

services["nova-api"].append("public")
services["nova-api"].append("internal")
```

```rust
fn main() {
    let mut services: HashMap<&str, Vec<&str>> = HashMap::new();

    services.entry("nova-api")
        .or_insert_with(Vec::new)
        .push("public");

    services.entry("nova-api")
        .or_insert_with(Vec::new)
        .push("internal");
}
```

На выходе `or_insert_with(Vec::new)` возвращает изменяемую ссылку на вектор внутри хешмапы. Поэтому можно сразу вызвать `push()`.

Итерации, опять же, не вызовут много вопросов:

```rust
for (svc, endpoint) in &services {
    println!("{}: {:?}", svc, endpoint);
}
```

В Python ключом словаря может быть не только строка. Например, можно использовать кортеж:

```python
endpoints = {}

endpoints[("10.0.0.1", 8774)] = "nova-api"
```

В Rust для этого можно использовать `struct`, которая реализует `PartialEq`, `Eq`, `Hash`, `Debug`:

```rust
use std::collections::HashMap;

#[derive(PartialEq, Eq, Hash, Debug)]
struct Endpoint {
    host: String,
    port: u16,
}

fn main() {
    let mut endpoints = HashMap::new();
    endpoints.insert(
        Endpoint { host: String::from("10.0.0.1"), port: 8774 },
        "nova-api",
    );

    println!("{:?}", endpoints);
}
```

Где:

* PartialEq значит, что экземпляры типа можно сравнивать через == или !=
* Eq равенство полное без странностей вроде NaN. Например, у чисел с плавающей точкой (f32, f64) есть PartialEq, но нет Eq, потому что в стандарте IEEE 754 NaN != NaN
* Hash для значения можно посчитать хеш. Без этого HashMap не поймёт, в какую корзину положить ключ
* Debug тип можно вывести через {:?}

В Python похожий пример выглядел бы короче, потому что кортежи уже умеют быть ключами, если внутри лежат хешируемые значения:

```python
endpoints = {
    ("10.0.0.1", 8774): "nova-api",
}
```

Но если попытаться использовать `list`, то получим TypeError

```python
endpoints = {}

endpoints[["10.0.0.1", 8774]] = "nova-api"  # TypeError
```

Потому что ключ словаря должен быть хешируемым и стабильным. Если ключ можно изменить после вставки, словарь может потерять его внутри себя, как таску в Jira после "давайте просто быстро поправим".

```rust
use std::collections::HashMap;

#[derive(PartialEq, Eq, Hash, Debug)]
struct Endpoint {
    host: String,
    port: u16,
}

fn main() {
    let mut endpoints = HashMap::new();

    let api = Endpoint {
        host: String::from("10.0.0.1"),
        port: 8774,
    };

    endpoints.insert(api, "nova-api");

    let lookup = Endpoint {
        host: String::from("10.0.0.1"),
        port: 8774,
    };

    match endpoints.get(&lookup) {
        Some(svc) => println!("найден: {}", svc),
        None => println!("не найден"),
    }
}
```

Ключ `api` переехал в хешмапу (move). При этом новый объект `lookup` находится по значению, потому что Endpoint реализует Eq и Hash.

## Структуры и классы данных

До этого мы использовали `HashMap` как аналог Python `dict`. Но есть важный момент: не каждый словарь должен быть словарем.

В Python часто начинают с такого:

```python
user = {
    "name": "Stas",
    "age": 38,
    "roles": ["systems-engineer", "systems-architect"],
}
```

А заканчивают когда в одном месте `age` число, в другом строка, а в третьем его вообще нет, потому что "ну там же временный словарик был". Если надо больше порядка, в Python можно использовать dataclass:

```python
from dataclasses import dataclass

@dataclass
class User:
    name: str
    age: int
    roles: list[str]

user = User(
    name="Stas",
    age=38,
    roles=["systems-engineer", "systems-architect"],
)

print(user)
```

Но Python остаётся Python. Type hints это подсказки, а не красные Егоры на этапе выполнения.

```python
user = User(
    name="Danil",
    age="38",
    roles=["team-lead"],
)
```

Заедет без каких-то проблем. Чтобы заставить Python соблюдать контракты нужны `mypy`, `pydantic` и другие 18+ игрушки.

В Rust для данных с заранее известным форматом есть struct:

```rust
struct User {
    name: String,
    age: u8,
    roles: Vec<String>,
}

fn main() {
    let user = User {
        name: String::from("Stas"),
        age: 38,
        roles: vec![
            String::from("systems-engineer"),
            String::from("systems-architect"),
        ],
    };
}
```

Если планируете выводить отладочную информацию, те не забываем про `#[derive(Debug)]`.

В Python `dataclass` изменяемый по-умолчанию, чтобы изменить это поведение надо использовать `@dataclass(frozen=True)`. В Rust наоборот. По умолчанию структура неизменяема, но используюя немного известной магии `mut` это поведение легко переопределяется:

```rust
#[derive(Debug)]
struct User {
    name: String,
    age: u8,
}

fn main() {
    let mut user = User {
        name: String::from("Stas"),
        age: 38,
    };

    user.age = 39;

    println!("{:?}", user);
}
```

Но чаще вы захотите создать новую версию структуры на основе старой, не ломая исходную:

```rust
#[derive(Debug)]
struct User {
    name: String,
    age: u8,
    active: bool,
}

fn main() {
    let user = User {
        name: String::from("Stas"),
        age: 38,
        active: true,
    };

    let updated = User {
        age: 39,
        ..user
    };

    println!("{:?}", updated);
}
```

Синтаксический сахар `..user` означает: "возьми остальные поля из user". При этом ..user перемещает поля, которые не переопределены явно. Если структура содержит String или другие типы не реализующие `Copy`, user после этого становится недоступен:

```rust
#[derive(Debug)]
struct User {
    name: String,
    age: u8,
    active: bool,
}

fn main() {
    let user = User {
        name: String::from("Stas"),
        age: 38,
        active: true,
    };

    let updated = User {
        age: 39,
        ..user
    };

    println!("{:?}", user);  // Ошибка: borrow of partially moved value: `user`
}
```

В итоге надо запомнить, что в Rust здравый смысл и компилятор заставляет выбрать инструмент под задачу и подписать контракт: кто владеет, что лежит внутри, можно ли менять.

| Задача                         | Python              | Rust            | Главное отличие                                             |
| ------------------------------ | ------------------- | --------------- | ----------------------------------------------------------- |
| Фиксированный набор однотипных | `tuple` / `list`    | `[T; N]`        | Размер часть типа, лежит в стеке                            |
| Набор разнотипных              | `tuple`             | `(T, U)`        | Доступ через `.0`, не `[0]`                                 |
| Динамический массив, стек      | `list`              | `Vec<T>`        | Один тип, move при присваивании, `get(i)` вместо IndexError |
| Очередь FIFO / двусторонняя    | `collections.deque` | `VecDeque<T>`   | Кольцевой буфер, O(1) с обоих концов                        |
| Ассоциативный массив           | `dict`              | `HashMap<K, V>` | Типизированные ключи и значения, `Option` при отсутствии    |
| Именованная структура данных   | `dataclass`         | `struct`        | Поля приватны по умолчанию, данные отдельно от поведения    |
