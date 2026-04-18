---
title: 'Rust для беженцев с Python - 1'
author: ["Stanislav"]
date: 2026-04-18T11:00:00+00:00
url: /2026/06/rust-for-python-refugees-1/
categories:
  - Tech
tags:
  - rust
  - python
---

# Rust для беженцев с Python - 1

Морозное январское утро. Салаты доедены, а организм всё ещё не понимает, какой сейчас год. Душа просит начать новый pet проект, которому никогда не суждено увидеть свет. И тут я узнаю, что вместо 
pip стильно модно молодежно теперь использовать написанный на Rust uv, а ядро pydantic v2 переписали на rust. Стоп, а зачем питоновским инструментам Rust?

Если убрать маркетинг, ответ довольно приземлённый. Python отлично справляется, пока:

* ты быстро пишешь код
* тебе важнее скорость разработки, а не выполнения
* ошибки можно поймать после "ща-ща докатим в прод до дедлайн и тестами обмажем"

Но есть класс задач, где это перестаёт работать комфортно:

* высокая стоимость ошибки
* производительность
* надежность

И в этих местах внезапно оказывается, что удобство Python начинает стоить ощутимых ресурсов.

## Привет, мир! и отличия синтаксиса

```python
print("hello, world!")
```


```rust
fn main() {
    println!("Hello, World!");
}
```

Отличие:

* в Python это выглядит как команда, интерпритатор читает файл и выполняет. В Rust требуется явно объявить точку входа `main()`
* в Python блоки выделяются отступами, в Rust заключено в `{...}`
* в Rust у println есть `!` - индикатор макроса. Помимо декларативных макросов `vec!`, `panic!` существуют еще процедурные, например `[derive(Debug, Serialize)]`
* в Python строка заканчивается переводом строки, в Rust `;`
* в Python код интерпритируется, в Rust компилируется и запускается `rustc hello.rs`

## Переменные и мутабельность 

```python
name = "Stas"
count = 0

for _ in range(3):
    count += 1
    print(f"Hello, {name}!")

print(f"Повторил {count} раза, понимаю как тяжело после нового года")
```

```rust
fn main() {
    let name = "Stas";
    let mut count = 0;

    for _ in 0..3 {
        count += 1;
        println!("Hello, {}!", name);
    }

    println!("Повторил {} раза, понимаю как тяжело после нового года", count);
}
```

* в Python переменные объявляются без типов. Присвоил и поехали. В Rust используется `let`, а тип либо указывается явно, либо выводится компилятором
* в Python переменные изменяемы по умолчанию. В Rust нет. Нужно явно написать `mut`. Тут первый культурный шок. Требуется подумать о том, будет ли эта переменная меняеться.
* в Python есть несколько способ форматирования строк. В Rust используется `{}` с опциональными позиционными `{0}, {1}`, именованными `{name}` аргументами или спецификаторами типа `{:?}`, `{:b}`, `{:x}`, `{:o}`

## Ownership и borrowing

```python
def greet(name):
    print(f"Hello, {name}!")

name = "Stas"
greet(name)
greet(name)
```

На Rust это выглядело бы так:

```rust
fn greet(name: String) {
    println!("Hello, {}!", name);
}

fn main() {
    let name = String::from("Stas");

    greet(name);
    greet(name); // ошибка компилятора
}
```

Если бы не borrow checker, с которым придется изрядно БОРОТЬСЯ в процессе написания кода и его отладки:

```bash
rustc hello.rs   

error[E0382]: use of moved value: `name`
 --> hello.rs:9:11
  |
6 |     let name = String::from("Stas");
  |         ---- move occurs because `name` has type `String`, which does not implement the `Copy` trait
7 |
8 |     greet(name);
  |           ---- value moved here
9 |     greet(name); // ошибка
  |           ^^^^ value used here after move
  |
note: consider changing this parameter type in function `greet` to borrow instead if owning the value isn't necessary
...
```

В Python нет необходимости думать кто владеет объектом и его использует. Ссылки, копии, память остаются под капотом и являются заботой сборщика мусора. 
В Rust у каждого объекта есть владелец и правила того, кто и как его может использовать. Значение `name` было перемещено в функцию `greet` и чтобы не терять переменную следует использовать заимствование `&`:

```rust
fn greet(name: &str) {
    println!("Hello, {}!", name);
}

fn main() {
    let name = String::from("Stas");

    greet(&name);
    greet(&name);
}
```

И тут появляется деталь, которая выглядит избыточной. Значение переменной через `&str` `let name = "Stas";` изменилось на `String` `let name = String::from("Stas");` 

* `&str` ты не владелец, а читатель. Строка лежит где-то в бинарнике, ты получаешь ссылку, ничего не аллоцируешь и не освобождаешь
* `String` когда появляется владелец. Память выделяется в куче, у строки есть владелец, при выходе из области память освобождается 

```
name (в стеке) -> [ ptr | len | capacity ] -> "Stas" (в куче)
```

Чтобы не терять переменную используется заимствование (borrow) и в коде появляется `&`, временный доступ. В функцию передается указатель, владельцем остается `main`. 
Но для Rust этого мало. Данные через обычную ссылку `&` нельзя изменить, требуется `&mut` при этом других ссылок в момент изменения быть не должно: либо много читающих `&`, либо один пищущий `&mut`. 

```rust
fn main() {
    let mut name = String::from("Stas");

    let r1 = &name;      // обычная ссылка
    let r2 = &mut name;  // ошибка

    println!("{}", r1);
    println!("{}", r2);
}
```

Все это проверяется в процессе компиляции, а не в момент выполнения кода:

```bash
rustc hello.rs

error[E0502]: cannot borrow `name` as mutable because it is also borrowed as immutable
 --> hello.rs:5:14
  |
4 |     let r1 = &name;      // обычная ссылка
  |              ----- immutable borrow occurs here
5 |     let r2 = &mut name;  // mutable ссылка — ошибка!
  |              ^^^^^^^^^ mutable borrow occurs here
6 |
7 |     println!("{}", r1);
  |                    -- immutable borrow later used here
```

При этом сама ссылка не может жить дольше, чем владелец:

```rust
fn main() {
    let r;

    {
        let name = String::from("Stas");
        r = &name;
    }

    println!("{}", r); // ошибка
}
```

```bash
rustc hello.rs

error[E0597]: `name` does not live long enough
 --> hello.rs:6:13
  |
5 |         let name = String::from("Stas");
  |             ---- binding `name` declared here
6 |         r = &name;
  |             ^^^^^ borrowed value does not live long enough
7 |     }
  |     - `name` dropped here while still borrowed
8 |
9 |     println!("{}", r); // ошибка
  |                    - borrow later used here
```

В Python отсутствие значений часть повседневных `AttributeError`:

```python
def get_user_name(user):
    return user.get("name")

user = {"id": 1}
name = get_user_name(user)

print(name.upper()) 
```

В Rust "забыли значение" уже тип:

```rust
fn get_user_name() -> Option<String> {
    None
}
```

Второй культурный шок последняя строка функции это и есть `return` в Python. А Option это либо `String` либо ничего. Под капотом это выглядит как или или: 

```rust
Some("Stas".to_string()) // есть значение
None                     // значения нет
```

Компилятор Rust на это мягонько намекнет:

```rust
fn get_user_name() -> Option<String> {
    None
}

fn main() {
    let name = get_user_name();

    println!("{}", name.to_uppercase()); // ошибка
}
```

```bash
rustc hello.rs

error[E0599]: no method named `to_uppercase` found for enum `Option<T>` in the current scope
 --> hello.rs:8:25
  |
8 |     println!("{}", name.to_uppercase()); // не скомпилируется
  |                         ^^^^^^^^^^^^ method not found in `Option<String>`
  |
note: the method `to_uppercase` exists on the type `String`
 --> /rustc/254b59607d4417e9dffbc307138ae5c86280fe4c\library\alloc\src\str.rs:466:5
help: consider using `Option::expect` to unwrap the `String` value, panicking if the value is an `Option::None`
  |
8 |     println!("{}", name.expect("REASON").to_uppercase()); // не скомпилируется
  |                        +++++++++++++++++

error: aborting due to 1 previous error
```

И ты просто обязан с этим что-то сделать:

```rust
fn main() {
    let name = get_user_name();

    match name {
        Some(n) => println!("{}", n.to_uppercase()),
        None => println!("Гость"),
    }
}
```

Либо, что используется чаще:

```rust
fn main() {
    let name = get_user_name().unwrap_or("Гость".to_string());
    println!("{}", name.to_uppercase());
}
```