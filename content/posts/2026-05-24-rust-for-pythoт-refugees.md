---
title: 'Rust для беженцев с Python - 4'
author: ["Stanislav"]
date: 2026-05-24T11:00:00+00:00
url: /2026/05/rust-for-python-refugees-4/
categories:
  - Tech
tags:
  - rust
  - python
---

После небольшого перерыва продолжаем с управляющими последовательностями.

## if/else

В Python if как диалог с понимающим барменом вечером пятницы. Видит твой уставший взгляд, палец на негрони, кивок без лишних вопросов:

```python
if user:
    send_email(user.email)
```

Пустой словарь? Значит нет пользователя.
`None`? Значит нет пользователя.
`user = ""`? Ну, получается, тоже пользователя не подвезли.

Когда проект небольшой такое поведение выглядит как забота и отсутствие overengineering'a. Когда проектик подрастает, то начинаешь ловить разнообразные спецэффекты с `None`, пустым словарем, словарем без `email`, строкой с `Null`, потому что парнерский сервис не прочитал контракт. 

Rust же врывается на танцпол с бюрократией и требует четкие инструкции кто, когда и куда свернет.

В `Python` `if` синонимично `попробуй угадай`. 

```python
def greet(user):
    if user:
        print(f"Hello, {user['name']}")
    else:
        print("Hello, guest")
```

Rust такое не любит:

```rust
fn main() {
    let user = "";

    if user {
        println!("Hello");
    }
}
```

В Rust "угадай" не компилируется. Тут условие должно быть `bool`. 

```rust
fn main() {
    let user = "";

    if !user.is_empty() {
        println!("Hello, {}", user);
    } else {
        println!("Hello, guest");
    }
}
```

* Пустая строка? `is_empty()`
* Пустой список? `is_empty()`
* Может быть значение? `Option`
* Может быть ошибка? `Result`
* Есть несколько вариантов? `match`

В Python `if` обычно вахтер на входе:

```python
service = "nova-api"

if service == "nova-api":
    port = 8774
elif service == "neutron-api":
    port = 9696
elif service == "keystone":
    port = 5000
else:
    port = None

print(port)
```

И все ок, пока кто-то через пару месяцев ниже по коду не вызовет `connect(port)` для `barbican-api` и `None` уезжает в функцию, которая ждала порт, а не отсутствие развернутого сервиса. 

Rust требует быть честнее:

```rust
fn port_for(service: &str) -> Option<u16> {
    if service == "nova-api" {
        Some(8774)
    } else if service == "neutron-api" {
        Some(9696)
    } else if service == "keystone" {
        Some(5000)
    } else {
        None
    }
}
```

Он не сильно лучше Python кода с точки зрения здравого смысла, но он **заставит** вызывающую функцию обработать `Some`, `None`, или взять ответственность с `unwrap()`.

# match и enum 

Настоящая разница начинается там, где появляется `enum`:

```rust
enum Service {
    Nova,
    Neutron,
    Keystone,
}
```

Теперь сервис не строка, а один из заранее известных вариантов.

```rust
fn port(service: Service) -> u16 {
    match service {
        Service::Nova => 8774,
        Service::Neutron => 9696,
        Service::Keystone => 5000,
    }
}

fn main() {
    let service = Service::Nova;
    let port = port(service);

    println!("port = {}", port);
}
``` 

Функция не принимает строку. Она принимает Service.

* Невозможно случайно передать "nvoa-api".
* Невозможно передать пустую строку.
* Невозможно передать "keystone " с пробелом.
* Невозможно передать "nova-api" в одном месте и "nova" в другом.

Да, в Python тоже есть `from enum import Enum`, но проблема в том, что статический анализатор может начать возмущаться, если он есть, включен, настроен, не игнорируется в CI и все участники команды не считают его личным оскорблением. Но сам Python в рантайме спокойно выполнит вызов. Строка? Строка. Функция? Функция. Какие ваши доказательства.

Теперь представим, что мы все же решили добавить новый сервис:

```rust
enum Service {
    Nova,
    Neutron,
    Keystone,
    Barbican,
}
```

Это вызовет ошибку в компиляции и потребует во всех функциях обработать его явно. 

Формально можно использовать `_` в качестве мусорного ведра

```rust
fn port(service: Service) -> u16 {
    match service {
        Service::Nova => 8774,
        Service::Neutron => 9696,
        Service::Keystone => 5000,
        _ => 8080,
    }
}
}
```

Но в инфраструктурном коде вы не захотите так делать. А в таком вполне себе:

```rust
fn status(code: u16) -> &'static str {
    match code {
        200 => "ok",
        400..=499 => "client error",
        500..=599 => "server error",
        _ => "unknown",
    }
}
```

`&'static str` выглядит как будто бы я упал лицом на клавиатуру, но это не так.  `&str` это ссылка на строку, а `'static` значит, что строка живёт всё время работы программы. Строковые литералы вроде `"ok"` и `"unknown"` зашиты в бинарник, поэтому функция может вернуть ссылку на них. Мы не создаём новый `String`, не аллоцируем память и не передаём владение. Просто возвращаем указатель на текст, который и так никуда не денется.

## for

Для итераций мы привыкли использовать циклы for и while:

```python
services = [
    {"name": "nova-api", "port": 8774},
    {"name": "neutron-api", "port": 9696},
    {"name": "keystone", "port": 5000},
]

for service in services:
    print(service["name"], service["port"])
```

В Rust начинаются спецэффекты borrow checker'a:

```rust
enum ServiceKind {
    Nova,
    Neutron,
    Keystone,
}

struct Service {
    kind: ServiceKind,
    name: String,
    port: u16,
}

fn main() {
    let services = vec![
        Service {
            kind: ServiceKind::Nova,
            name: String::from("nova-api"),
            port: 8774,
        },
        Service {
            kind: ServiceKind::Neutron,
            name: String::from("neutron-api"),
            port: 9696,
        },
        Service {
            kind: ServiceKind::Keystone,
            name: String::from("keystone"),
            port: 5000,
        },
    ];

    for service in services {
        println!("{} -> {}", service.name, service.port);
    }

    println!("services count = {}", services.len());
}
```

```
36 |     println!("services count = {}", services.len());
   |                                     ^^^^^^^^ value borrowed here after move
```

Мы не просто проитерировались по сервисам, мы "съели" этот вектор. Элементы переезжают внутрь цикла, а после цикла services больше недоступен.

Так что итерируясь вы должны помнить про то, что либо вы забираете владение:

```rust
for service in services {
    println!("{} -> {}", service.name, service.port);
}
```

либо читаете по ссылке:

```rust
for service in &services {
    println!("{} -> {}", service.name, service.port);
}
```

либо меняете по изменяемой ссылке:

```
for service in &mut services {
    service.name.push_str("-internal");
}
```

Если нужно изменить данные, то, опять же, держим в памяти `mut`:

```rust
fn main() {
    let mut endpoints = vec![
        String::from("http://10.0.0.10:8774"),
        String::from("http://10.0.0.11:9696"),
        String::from("http://10.0.0.12:5000"),
    ];

    for endpoint in &mut endpoints {
        endpoint.push_str("/healthcheck");
    }

    println!("{:?}", endpoints);
}
```

То есть мы явно говорим: хочу поменять элементы внутри коллекции. 

Если надо достать индекс, то достаем знакомый по python `enumerate`:

```rust
for (index, service) in services.iter().enumerate() {
    println!("{}: {}", index, service);
}
```

Где `iter()` отдаёт ссылки на элементы. Если же нужно забрать значения, есть into_iter():

```rust
for service in services.into_iter() {
    println!("{}", service);
}
```

А если поменять `iter_mut()`:

```rust
fn main() {
    let mut services = vec![
        String::from("nova-api"),
        String::from("neutron-api"),
        String::from("keystone"),
    ];

    for service in services.iter_mut() {
        service.push_str("-internal");
    }

    println!("{:?}", services);
}
```

Это же и естественная замена `list comprehension`.

### break и continue

За исключением синтаксического сахара все привычно:

```rust
fn main() {
    let services = vec!["nova-api", "neutron-api", "keystone"];

    for service in &services {
        if *service == "neutron-api" {
            continue;
        }

        if *service == "keystone" {
            break;
        }

        println!("checking {}", service);
    }
}
```

Почему в синтаксис языка еще звездочка `*` просыпалась? Когда мы пишем for service in &services, мы получаем ссылку на элемент. Если вектор хранит &str, то ссылка на него будет &&str или ссылка на ссылку. Если слишком много сахара, то можно использовать `pattern destructuring`:

```rust
fn main() {
    let services = vec!["nova-api", "neutron-api", "keystone"];

    for &service in &services {
        if service == "neutron-api" {
            continue;
        }

        if service == "keystone" {
            break;
        }

        println!("checking {}", service);
    }
}
```

Выходить из вложенных циклов удобнее через метки:

```rust
struct Endpoint {
    interface: String,
    url: String,
}

struct Service {
    name: String,
    endpoints: Vec<Endpoint>,
}

fn main() {
    let services = vec![
        Service {
            name: String::from("nova-api"),
            endpoints: vec![
                Endpoint {
                    interface: String::from("public"),
                    url: String::from("https://public.example.com:8774"),
                },
                Endpoint {
                    interface: String::from("internal"),
                    url: String::from("http://10.0.0.10:8774"),
                },
            ],
        },
        Service {
            name: String::from("keystone"),
            endpoints: vec![
                Endpoint {
                    interface: String::from("public"),
                    url: String::from("https://public.example.com:5000"),
                },
            ],
        },
    ];

    'search: for service in &services {
        for endpoint in &service.endpoints {
            if endpoint.interface == "internal" {
                println!("found internal endpoint: {}", endpoint.url);
                break 'search;
            }
        }
    }
}
```

Метка `'search` позволяет выйти не только из внутреннего цикла, а сразу из внешнего без `found = True` и отдельного условия `if found: break`.

## while

While в Rust скучный и это комплимент:

```rust
fn main() {
    let mut retries = 3;

    while retries > 0 {
        println!("trying to reach API endpoint...");
        retries -= 1;
    }

    println!("giving up");
}
```

while True, тоже, в общем-то:

```rust
loop {
    println!("работаем, братья");
}
```

Для выхода из `loop` можно использовать классический подход Python:

```rust
fn main() {
    let mut attempts = 0;

    let status = loop {
        attempts += 1;

        println!("checking endpoint, attempt {}", attempts);

        if attempts == 3 {
            break "endpoint is alive";
        }

        if attempts > 5 {
            break "endpoint is dead";
        }
    };

    println!("{}", status);
}
```

## any() / all()

Тут, в общем, тоже без приключений:

```python
has_internal = any(
    endpoint["interface"] == "internal"
    for endpoint in endpoints
)
```

Элегантно превращается в:

```rust
let has_internal = services
    .iter()
    .flat_map(|service| service.endpoints.iter())
    .any(|endpoint| endpoint.interface == "internal");
```

```rust
let all_have_url = services
    .iter()
    .flat_map(|service| service.endpoints.iter())
    .all(|endpoint| !endpoint.url.is_empty());
```