# Rust 进阶

## 生命周期进阶

```rust
// 生命周期省略规则
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }
    &s[..]
}

// 复杂生命周期
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

## Trait 系统

```rust
trait Summary {
    fn summarize(&self) -> String;
    
    // 默认实现
    fn preview(&self) -> String {
        format!("{}...", &self.summarize()[..50])
    }
}

struct Article {
    title: String,
    content: String,
}

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{}: {}", self.title, self.content)
    }
}

// Trait 作为参数
fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}
```

## 异步编程

```rust
use tokio;

#[tokio::main]
async fn main() {
    let result = fetch_data().await;
    println!("{}", result);
}

async fn fetch_data() -> String {
    tokio::time::sleep(Duration::from_secs(1)).await;
    "Data fetched".to_string()
}
```

## 宏

```rust
// 声明宏
macro_rules! vec {
    ( $( $x:expr ),* ) => {
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}

let v = vec![1, 2, 3];
```

## unsafe Rust

```rust
unsafe {
    let raw = &mut x as *mut i32;
    *raw = 42;
}

// 安全抽象
fn split_at_mut(slice: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = slice.len();
    assert!(mid <= len);
    
    let ptr = slice.as_mut_ptr();
    unsafe {
        (
            std::slice::from_raw_parts_mut(ptr, mid),
            std::slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}
```
