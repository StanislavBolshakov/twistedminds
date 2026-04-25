---
title: 'Rust для беженцев с Python - 2'
author: ["Stanislav"]
date: 2026-04-24T11:00:00+00:00
url: /2026/04/rust-for-python-refugees-2/
categories:
  - Tech
tags:
  - rust
  - python
---

В прошлый раз мы познакомились с синтаксисом, узнали, что переменные по умолчанию неизменяемы, и посетили психотерапевта после знакомства с Borrow Checker. Теперь пора выйти за рамки "Hello, Stas" и разобраться, как в Rust устроены типы, отличные от `String` и `&str`. Без этого дальше не поедем. 

## Числовые типы 

Но есть и хорошие новости. Не всегда в Rust придется сражаться с Borrow Сhecker. Иногда всё почти как в Python, только надо чуть-чуть докрутить. 

В Python число это просто число:

```python
count = 69
temperature = -5.3
index = 0
```

Все это `int` или `float`. Python сам решает, сколько байт выделить, пока ты сидишь и программировавуешь. В Rust такой роскоши нет, надо выбрать тип:

```rust
fn main() {
    let count: u8 = 69;
    let temperature: f64 = -5.3;
    let index: usize = 0;
}
```

В Rust целое ведро числовых типов. `i8`, `i16`, `i32`, `i64`, `i128` с знаком и `u8`, `u16`, `u32`, `u64`, `u128` без. Буква говорит о знаке, цифра о размере в битах. Помимо этого есть `isize` и `usize`.

| Тип | Диапазон | Python-аналог |
|-----|----------|---------------|
| `i8` | -128..127 | нет |
| `i32` | ~-2 млрд..2 млрд | `int` |
| `i64` | подойдет для ипотеки в 2026 | `int` |
| `u8` | 0..255 | нет |
| `usize` | зависит от архитектуры | нет |
| `f32`, `f64` | числа с плавающей точкой | `float` |

```rust
fn main() {
    let small: u8 = 255;
    let big: u64 = 18_123_456_789_10;
    let precise: f64 = 3.14;
}
```

Более того, в отличии от `String` такой код вполне себе рабочий:

```rust
fn main() {
    let i: u8 = 5;
    let x = i;
    println!("Число равно {}", i)
}
```

Почему? u8 один байт, скопировать его дешевле, чем объяснять человеку почему в stdout нельзя вывести пятерку. Никакой драмы, никто никуда не переехал, чемоданы не собрал и не выяснял отношения. Было `i = 5`, стало `i = 5, x = 5`, потому что числовые, булевы и `char` типы реализуют trait `Copy`. Если очень грубо, `trait` это описание поведения. Не конкретная реализация, не объект, не класс, а именно обещание:

> тип, который реализует этот trait, умеет делать вот это

Теперь занырнем в числовые типы глубже. А что будет, если переполнить `u8`?

```rust
fn main() {
    let mut count: u8 = 255;
    count += 1; // ошибка в debug, wrap в release
}
```

В Rust компилятор не даст вам забыть о диапазоне типа.

```bash
rustc hello.rs
.\hello.exe

thread 'main' (25268) panicked at hello.rs:3:5:
attempt to add with overflow
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

$ rustc -C opt-level=3  hello.rs
$.\hello.exe
count = 0
```

Поведение при переполнении можно изменить:

```rust
fn main() {
    let a: u8 = 255;

    let b = a.checked_add(1);
    let c = a.wrapping_add(1);
    let d = a.saturating_add(1);
    let (e, overflowed) = a.overflowing_add(1);

    println!("checked:    {:?}", b);           // None
    println!("wrapping:   {}", c);             // 0
    println!("saturating: {}", d);             // 255
    println!("overflowing: {} (overflowed={})", e, overflowed);  // 0 true
}
```

| Метод | Возвращает | Когда использовать |
|-------|------------|-------------------|
| `checked_add` | `Option<T>` | `None` при переполнении |
| `wrapping_add` | `T` | арифметика, хеши |
| `saturating_add` | `T` | остается на границе |
| `overflowing_add` | `(T, bool)` | ручная проверка, где нужно и значение и факт переполнения |

Ещё один момент: `usize` (и `isize`) это размер указателя на текущей архитектуре. 64 бита на x86_64, 32 на старых системах. Именно `usize` требуется для индексации массивов, и именно его возвращают методы вроде `.len()`. Это важно запомнить, потому что так поедет:

```rust
fn main() {
    let numbers = [10, 20, 30];
    let i: usize = 1;

    println!("{}", numbers[i]); // 20
    println!("{}", numbers.len()); // 3
}
```

А так нет:

```rust
fn main() {
    let numbers = [10, 20, 30];
    let i: u8 = 1;

    println!("{}", numbers[i]);
}
```

```bash
rustc hello.rs                                                                                                                                     
error[E0277]: the type `[{integer}]` cannot be indexed by `u8`
 --> hello.rs:5:28
  |
5 |     println!("{}", numbers[i]);
  |                            ^ slice indices are of type `usize` or ranges of `usize`
  |
  = help: the trait `SliceIndex<[{integer}]>` is not implemented for `u8`
```

Надо или подумать заранее или привести через `i as usize` или полагаться на встроенную логику приведения к типу (`type inference`)

## Type inference и type hints

В Python c 3.5 появились `type hints`:

```python
def greet(name: str, age: int) -> str:
    return f"Привет, {name}! You are {age} already, kindly stop shitpost on the Internets."

print(greet("Stas", 38))
```

Это добровольная аннотация. Интерпретатор её игнорирует, mypy проверяет по желанию. Можно замнить на `greet("Stas", "Thirty-eight")` и ошибкой это не будет, все запустится. А ошибку поймаете (может быть) в проде.

В Rust тип тоже можно не писать явно, компилятор выведет его сам:

```rust
fn main() {
    let count = 69;        // i32 по умолчанию
    let name = "Stas";     // &str
    let pi = 3.14;         // f64 по умолчанию
    let numbers = [10, 20, 30];
    let i = 1;              // тип пока не определен

    println!("{}", numbers[i]); // i стал usize
}
```

Inference - способность компилятора однозначно определить тип из контекста. Если контекста недостаточно, он потребует четких инструкций:

```rust
fn main() {
    let guess = "69".parse(); // ошибка
}
```

Потому что `.parse()` может вернуть `i32`, `u64`, `f32` что угодно, реализующее `FromStr`. Компилятор не будет угадывать. Если считаешь, что ошибки тут быть не может, то смело доставай `unwrap()`

```rust
fn main() {
    let guess: u32 = "69".parse().unwrap();
    println!("The answer to the Ultimate Question of Life, The Universe, and Everything is: {}", guess);
}
```

или короче через turbofish `::<>` с миллитриком в губки:

```rust
let guess = "69".parse::<u32>().unwrap();
```

Но будь готов к последствиям:

```rust
fn main() {
    let guess: u32 = "not a number".parse().unwrap();
    println!("The answer to the Ultimate Question of Life, The Universe, and Everything is: {}", guess);
}
```

Приложение компилируется, но упадет в панику:

```bash
.\hello.exe

thread 'main' (40640) panicked at hello.rs:2:45:
called `Result::unwrap()` on an `Err` value: ParseIntError { kind: InvalidDigit }
```

И тут в здание заходит `Result`, который возвращает `.parse()`. В прошлой статье мы познакомились с `Option` где либо все, либо ничего. Но с `.parse()` ситуация иная: значения просто не может не быть. Для таких ситуаций и существует `Result` - действие не получилось и вот почему:

```rust
Ok(value)
Err(error)
```

Unwrap хорош для прототипирования, но когда на вход может залететь все, что угодно от валидного JSON, до лол кек чебурек, лучше написать чуть больше кода:

```rust
fn main() {
    let guess = "not a number".parse::<u32>();

    match guess {
        Ok(n) => println!("Число, получается: {}", n),
        Err(e) => println!("Не получилось распарсить: {}", e),
    }
}
```

Спорить что лучше: это или отлавливать `TypeError` с `ValueError` решать тебе. 

## Pattern Matching

В примере выше появился `match`. Match это швейцарский нож, который умеет гораздо больше. 

```rust
fn main() {
    let temp: i8 = -5;

    match temp {
        t if t < -10 => println!("Холоднее чем в Сибири, потому что у нас влажность высокая"),
        -10..=0 => println!("Холодно"),
        1..=15 => println!("Тепло"),
        _ => println!("Нерелевантно для Петербурга"),
    }
}
```

В Python это выглядело бы так:

```python
temp = -5

if temp < -10:
    print("Мороз, потому что у нас влажность высокая")
elif -10 <= temp <= 0:
    print("Холодно")
elif 1 <= temp <= 15:
    print("Тепло")
else:
    print("Нерелевантно для Петербурга")
```

Если убрать `_`, а диапазоны не покроют все значения `i8`, компилятор не намекнёт, а выйдет в окно с `patterns not covered` так как match не исчерпывающий. 