# Rust 基础

## 什么是 Rust？

Rust 是一门系统编程语言，注重安全性、并发性和性能。

## 快速开始

```rust
fn main() {
    println!("Hello, world!");
}
```

## 变量和类型

```rust
// 不可变变量
let x = 5;
let name = "John";

// 可变变量
let mut y = 10;
y += 1;

// 类型注解
let age: i32 = 25;
let height: f64 = 1.75;

// 常量
const MAX_POINTS: u32 = 100_000;
```

## 所有权系统

```rust
// 所有权规则
let s1 = String::from("hello");
let s2 = s1;  // s1 被移动，不再有效

// 克隆
let s3 = s2.clone();

// 引用（借用）
let s4 = String::from("hello");
let len = calculate_length(&s4);  // 不可变借用
println!("'{}' 的长度是 {}", s4, len);

fn calculate_length(s: &String) -> usize {
    s.len()
}
```

## 结构体

```rust
struct User {
    username: String,
    email: String,
    active: bool,
}

impl User {
    fn new(username: &str, email: &str) -> User {
        User {
            username: String::from(username),
            email: String::from(email),
            active: true,
        }
    }
    
    fn greet(&self) -> String {
        format!("Hello, {}!", self.username)
    }
}

let user = User::new("john", "john@example.com");
println!("{}", user.greet());
```

## 枚举和模式匹配

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
}

fn process_message(msg: Message) {
    match msg {
        Message::Quit => println!("Quit"),
        Message::Move { x, y } => println!("Move to ({}, {})", x, y),
        Message::Write(text) => println!("Write: {}", text),
    }
}
```

## 错误处理

```rust
use std::fs::File;
use std::io::Read;

fn read_file(path: &str) -> Result<String, std::io::Error> {
    let mut file = File::open(path)?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents)
}

match read_file("file.txt") {
    Ok(contents) => println!("{}", contents),
    Err(e) => println!("Error: {}", e),
}
```

## 并发

```rust
use std::thread;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("Spawned thread: {}", i);
            thread::sleep(std::time::Duration::from_millis(1));
        }
    });
    
    for i in 1..5 {
        println!("Main thread: {}", i);
        thread::sleep(std::time::Duration::from_millis(1));
    }
    
    handle.join().unwrap();
}
```