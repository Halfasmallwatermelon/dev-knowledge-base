# Rust 进阶技术文档

> 适用对象：已掌握 Rust 基础语法、所有权、闭包、模块系统的开发者。  
> 目标：系统梳理 Rust 高级特性，提供可直接运行的工程级示例与最佳实践。

---

## 1. 概述

Rust 的进阶学习核心围绕**内存安全、类型系统、并发模型与工具链生态**展开。进阶路线建议如下：

| 阶段 | 重点 | 产出能力 |
|------|------|----------|
| 第一阶段 | 所有权、生命周期、借用检查器 | 写出无数据竞争、零运行时开销的代码 |
| 第二阶段 | 泛型、Trait、动态/静态分发 | 设计可扩展的类型抽象与插件架构 |
| 第三阶段 | 错误处理、异步编程 | 构建生产级 CLI、服务端与流式处理 |
| 第四阶段 | 宏系统、Unsafe、FFI | 编写高性能底层库、绑定外部 C/C++/SIMD |
| 第五阶段 | 性能调优、测试、CI/CD | 交付可维护、可观测、可发布的工程 |

### 环境准备代码

```toml
# Cargo.toml (进阶实验项目)
[package]
name = "rust-advanced-lab"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1", features = ["full"] }
thiserror = "1"
anyhow = "1"
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "perf_bench"
harness = false
```

---

## 2. 所有权系统进阶

### 2.1 生命周期注解与 HRTB

Rust 编译器默认无法推断复杂引用关系时，需显式标注生命周期。高阶临时生命周期（HRTB）用于描述“对任意生命周期均成立”的约束：

```rust
trait Apply<'a> {
    fn apply(&self, input: &'a str) -> String;
}

// HRTB: 对任意 'a，函数指针都满足 Apply<'a>
fn call_apply<F>(applier: &F, input: &str) -> String
where
    F: for<'a> Apply<'a>,
{
    applier.apply(input)
}
```

### 2.2 NLL（Non-Lexical Lifetimes）

NLL 允许编译器根据实际使用点而非代码块作用域来终止借用。这使以下模式合法：

```rust
let x = String::from("hello");
let y = &x;
println!("{}", y);
// NLL: y 在此处结束生命周期，后续可重新借用 x
let z = &mut x;
z.push_str(" world");
println!("{}", z);
```

### 2.3 借用检查器高级模式：Interior Mutability

当需要共享不可变引用但内部可变时，使用 `RefCell`、`Mutex`、`Atomic` 等类型。

#### 完整可运行示例

```rust
use std::cell::RefCell;
use std::collections::HashMap;

struct Cache {
    data: RefCell<HashMap<String, i64>>,
}

impl Cache {
    fn new() -> Self {
        Cache {
            data: RefCell::new(HashMap::new()),
        }
    }

    fn get(&self, key: &str) -> Option<i64> {
        self.data.borrow().get(key).copied()
    }

    fn insert(&self, key: String, value: i64) {
        self.data.borrow_mut().insert(key, value);
    }
}

fn main() {
    let cache = Cache::new();
    cache.insert("alpha".to_string(), 10);
    cache.insert("beta".to_string(), 20);

    println!("alpha = {}", cache.get("alpha").unwrap());
    
    // RefCell 在运行时检查借用规则
    let mut map = cache.data.borrow_mut();
    map.insert("gamma".to_string(), 30);
    drop(map); // 必须主动释放或让作用域结束
    
    println!("gamma = {}", cache.get("gamma").unwrap());
}
```

---

## 3. 泛型与 trait

### 3.1 高级 Trait Bound

使用 `where` 子句分离定义与约束，支持多约束、`Sized`、`+ Send + Sync` 等线程安全标记。

### 3.2 关联类型 vs 泛型参数

| 特性 | 关联类型 | 泛型参数 |
|------|----------|----------|
| 语法 | `type Item;` | `type Item<T>;` |
| 实例化 | 每个实现固定一个 | 每次使用可不同 |
| 适用场景 | 协议/迭代器 | 容器/多态算法 |

### 3.3 Trait Object 与 `impl Trait`

- `dyn Trait`：动态分发，运行时通过虚表调用，支持多态存储。
- `impl Trait`：静态分发，编译器内联展开，类型必须单一。

#### 完整可运行示例

```rust
use std::fmt;

trait Shape {
    fn area(&self) -> f64;
    fn describe(&self) -> String;
}

struct Circle { r: f64 }
struct Rectangle { w: f64, h: f64 }

impl Shape for Circle {
    fn area(&self) -> f64 { std::f64::consts::PI * self.r * self.r }
    fn describe(&self) -> String { format!("Circle(r={})", self.r) }
}

impl Shape for Rectangle {
    fn area(&self) -> f64 { self.w * self.h }
    fn describe(&self) -> String { format!("Rect(w={}, h={})", self.w, self.h) }
}

// 返回 impl Trait：静态分发，性能最优
fn draw(shape: &impl Shape) -> String {
    format!("Drawing {} with area {:.2}", shape.describe(), shape.area())
}

// 动态分发：可存储不同类型
fn print_all(shapes: &[&dyn Shape]) {
    for s in shapes {
        println!("{}", draw(s));
    }
}

fn main() {
    let c = Circle { r: 5.0 };
    let r = Rectangle { w: 3.0, h: 4.0 };

    println!("{}", draw(&c));
    
    let shapes: Vec<&dyn Shape> = vec![&c, &r];
    print_all(&shapes);
}
```

---

## 4. 错误处理进阶

### 4.1 自定义错误类型

生产代码推荐使用枚举 + `thiserror` 派生，保留错误上下文与来源链。

### 4.2 `anyhow` 适用场景

`anyhow::Error` 适合应用层入口，自动包装错误链；库代码应暴露具体错误类型。

### 4.3 错误转换链

`From<T> for E` + `?` 运算符自动向上游传播并携带 `source()` 信息。

#### 完整可运行示例

```rust
use thiserror::Error;
use std::fs;
use std::io;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("IO error: {0}")]
    Io(#[from] io::Error),

    #[error("Parse error at line {line}: {msg}")]
    Parse { line: usize, msg: String },

    #[error("Config not found: {path}")]
    ConfigMissing { path: String },
}

fn read_config(path: &str) -> Result<String, AppError> {
    let content = fs::read_to_string(path)?; // 自动转换为 AppError::Io
    if content.is_empty() {
        return Err(AppError::ConfigMissing {
            path: path.to_string(),
        });
    }
    Ok(content)
}

fn parse_first_line(content: &str) -> Result<i32, AppError> {
    content
        .lines()
        .next()
        .ok_or_else(|| AppError::Parse { line: 0, msg: "empty file".into() })?
        .trim()
        .parse::<i32>()
        .map_err(|e| AppError::Parse { line: 1, msg: e.to_string() })
}

fn main() -> Result<(), AppError> {
    let config = read_config("demo.txt")?;
    let value = parse_first_line(&config)?;
    println!("Parsed value: {value}");
    Ok(())
}
```

> 运行前请创建 `demo.txt` 并写入首行数字（如 `42`）。

---

## 5. 异步编程

### 5.1 `async/await` 与 Future

`async fn` 会被编译为返回实现 `Future<Output = T>` 的匿名类型。`.await` 仅在异步上下文或 `#[tokio::main]` 中可用。

### 5.2 Tokio Runtime

- `current_thread`：轻量单线程，适合 CPU 密集或简单任务
- `multi_thread`：多线程调度，适合 I/O 密集型服务

### 5.3 异步 Stream

使用 `tokio_stream` 或 `futures::stream` 处理无限/分批数据。

#### 完整可运行示例

```rust
use tokio::time::{sleep, Duration};
use tokio_stream::{StreamExt, iter};

async fn fetch_user(id: u64) -> String {
    sleep(Duration::from_millis(50)).await;
    format!("User-{id}")
}

async fn process_batch(ids: Vec<u64>) {
    let stream = iter(ids.into_iter().map(fetch_user));
    
    // 并发限制为 3
    let mut stream = stream.buffer_unordered(3);
    while let Some(name) = stream.next().await {
        println!("Processed: {name}");
    }
}

#[tokio::main(flavor = "current_thread")]
async fn main() {
    let ids = vec![1, 2, 3, 4, 5];
    process_batch(ids).await;
}
```

---

## 6. 宏系统

### 6.1 声明宏 (`macro_rules!`)

适用于语法转换、重复模式、条件编译。注意 `$()*` 重复语法与 `$var:tt` 片段类型。

### 6.2 过程宏

分为三类：
- `#[proc_macro]`：函数式宏
- `#[proc_macro_derive]`：派生宏
- `#[proc_macro_attribute]`：属性宏

过程宏必须在独立 crate 中定义，并通过 `proc-macro = true` 声明。

#### 完整可运行示例

```rust
// 声明宏：自动生成带时间戳的日志调用
macro_rules! log_info {
    ($($arg:tt)*) => {
        println!("[INFO] {}", format_args!($($arg)*));
    };
}

// 声明宏：安全获取结构体字段（仅演示语法）
macro_rules! define_counter {
    ($name:ident, $init:expr) => {
        struct $name {
            count: $init,
        }
        impl $name {
            fn new() -> Self {
                $name { count: 0 }
            }
            fn increment(&mut self) { self.count += 1; }
            fn get(&self) -> i32 { self.count }
        }
    };
}

define_counter!(ClickCounter, i32);

fn main() {
    log_info!("System started, id={}", 1001);

    let mut counter = ClickCounter::new();
    counter.increment();
    counter.increment();
    println!("Count: {}", counter.get());
}
```

> 注：过程宏完整示例需拆分为 `my_lib` 和 `my_lib_macros` 两个 crate，此处因单文件可运行要求使用声明宏替代展示。

---

## 7. unsafe Rust

### 7.1 裸指针与内存安全边界

`*const T` / `*mut T` 不携带所有权语义，解引用必须放在 `unsafe` 块中。Rust 仍要求遵守别名规则。

### 7.2 FFI 与 `#[repr(C)]`

跨语言调用必须保证内存布局一致。使用 `CString`/`CStr` 避免悬垂指针。

### 7.3 `MaybeUninit` 与手动生命周期

替代不安全的 `uninitialized()`，用于零初始化或延迟构造。

#### 完整可运行示例

```rust
use std::mem::MaybeUninit;
use std::ffi::{CString, CStr};

// 模拟 C 函数签名
extern "C" {
    fn c_add(a: i32, b: i32) -> i32;
}

// 提供 C 兼容实现供 FFI 调用
#[no_mangle]
pub extern "C" fn c_add(a: i32, b: i32) -> i32 {
    a.wrapping_add(b)
}

fn safe_divide(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 { None } else { Some(a / b) }
}

fn main() {
    // FFI 调用
    let result = unsafe { c_add(10, 32) };
    println!("c_add(10, 32) = {result}");

    // MaybeUninit：延迟初始化
    let mut val: MaybeUninit<String> = MaybeUninit::uninit();
    unsafe {
        val.write(String::from("delayed"));
        let initialized = val.assume_init();
        println!("Initialized: {initialized}");
    }

    // 安全封装裸指针用法
    let s = CString::new("hello").unwrap();
    let c_str: &CStr = unsafe { CStr::from_ptr(s.as_ptr()) };
    println!("C string: {:?}", c_str.to_str().unwrap());
}
```

---

## 8. 性能优化

### 8.1 零成本抽象

Rust 的泛型、Trait、闭包在编译期展开，不引入额外运行时开销。

### 8.2 内联与冷路径

- `#[inline]`：提示编译器内联
- `#[inline(never)]` / `#[cold]`：避免热路径膨胀

### 8.3 性能分析工具

| 工具 | 用途 |
|------|------|
| `cargo bench` + Criterion | 基准测试 |
| `perf` / `flamegraph` | CPU 火焰图 |
| `valgrind --tool=callgrind` | 调用开销分析 |
| `cargo clippy -W clippy::all` | 静态性能警告 |

#### 完整可运行示例

```rust
use std::hint::black_box;

#[inline]
fn fast_multiply(a: i64, b: i64) -> i64 {
    a.wrapping_mul(b)
}

#[inline(never)]
#[cold]
fn slow_log(msg: &str) {
    eprintln!("[SLOW] {msg}");
}

fn compute(data: &[i64]) -> i64 {
    data.iter().fold(1i64, |acc, &x| fast_multiply(acc, x))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn benchmark_inline() {
        let data: Vec<i64> = (1..=1000).collect();
        let start = std::time::Instant::now();
        for _ in 0..100_000 {
            black_box(compute(&data));
        }
        let elapsed = start.elapsed();
        println!("compute 100k iterations: {:?}", elapsed);
    }
}

fn main() {
    let data: Vec<i64> = (1..=100).collect();
    println!("Product: {}", compute(&data));
    slow_log("operation completed");
}
```

---

## 9. 代码示例：综合实战

以下示例整合所有权、Trait、错误处理、异步、宏与性能优化，可直接运行。

```rust
use std::cell::RefCell;
use std::collections::HashMap;
use std::fmt;
use thiserror::Error;
use tokio::time::{sleep, Duration};

// --- 错误定义 ---
#[derive(Error, Debug)]
pub enum ServiceError {
    #[error("database unavailable: {0}")]
    DbDown(String),
    #[error("invalid input: {0}")]
    InvalidInput(String),
}

// --- Trait 抽象 ---
trait Repository {
    fn find(&self, id: u64) -> Option<String>;
    fn save(&self, id: u64, value: String);
}

// --- 内存实现（RefCell 实现内部可变） ---
struct InMemoryRepo {
    store: RefCell<HashMap<u64, String>>,
}

impl InMemoryRepo {
    fn new() -> Self { Self { store: RefCell::new(HashMap::new()) } }
}

impl Repository for InMemoryRepo {
    fn find(&self, id: u64) -> Option<String> { self.store.borrow().get(&id).cloned() }
    fn save(&self, id: u64, value: String) { self.store.borrow_mut().insert(id, value); }
}

// --- 声明宏：统一响应格式 ---
macro_rules! respond {
    ($status:expr, $body:expr) => {
        format!("HTTP/1.1 {} {}\r\nContent-Length: {}\r\n\r\n{}", $status, $status, $body.len(), $body)
    };
}

// --- 异步服务 ---
async fn handle_request(repo: &InMemoryRepo, id: u64) -> Result<String, ServiceError> {
    sleep(Duration::from_millis(20)).await;
    repo.find(id)
        .ok_or_else(|| ServiceError::InvalidInput(format!("ID {id} not found")))
}

#[tokio::main]
async fn main() -> Result<(), ServiceError> {
    let repo = InMemoryRepo::new();
    repo.save(1, "Alice".to_string());
    repo.save(2, "Bob".to_string());

    let id = 1;
    match handle_request(&repo, id).await {
        Ok(name) => println!("{}", respond!("200 OK", name)),
        Err(e) => eprintln!("{}", respond!("404 Not Found", e.to_string())),
    }

    Ok(())
}
```

---

## 10. 最佳实践

### 10.1 项目结构

```text
my-project/
├── Cargo.toml              # workspace 根
├── crates/
│   ├── core/               # 领域模型与 trait
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── service/            # 业务逻辑
│   └── app/                # CLI / Web 入口
├── tests/                  # 集成测试
├── benches/                # 性能测试
└── .github/workflows/ci.yml
```

### 10.2 测试策略

- 单元测试：`#[cfg(test)] mod tests`，覆盖边界条件
- 集成测试：`tests/` 目录，仅通过公开 API
- Doc Tests：`///` 注释中的代码块自动执行
- Fuzz：`cargo-fuzz` 发现内存与逻辑漏洞

### 10.3 CI/CD 集成

#### 完整可运行配置示例

```yaml
# .github/workflows/ci.yml
name: Rust CI

on: [push, pull_request]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy
      - run: cargo fmt --check
      - run: cargo clippy -- -D warnings
      - run: cargo test --all-targets
      - run: cargo build --release
```

---

## 11. 常见问题

### 11.1 常见编译错误与解决方案

| 错误信息 | 原因 | 解决方案 |
|----------|------|----------|
| `borrowed value does not live long enough` | 引用生命周期不匹配 | 添加 `'a` 标注，或使用 `String`/`Cow` 转移所有权 |
| `cannot move out of borrowed content` | 尝试移动借用数据的 owned 值 | 使用 `.clone()` 或改为接收 `&str`/`&[T]` |
| `the trait bound is not satisfied` | 泛型缺少必要约束 | 在 `where` 子句补充 `T: Clone + Send` 等 |
| `future cannot be sent between threads safely` | `Send` 约束缺失 | 确保 `Future` 内不含 `Rc`/非 `Sync` 类型 |
| `mismatched types: expected ..., found ...` | 类型推断失败 | 使用类型注解或 turbofish `::<T>` |

### 11.2 修复示例

```rust
// ❌ 错误：尝试从借用中移动 String
fn bad(s: &String) -> String {
    s.clone() // 正确做法：显式 clone
}

// ✅ 推荐：使用 Cow 避免不必要的分配
use std::borrow::Cow;

fn good(input: &str) -> Cow<str> {
    if input.contains("TODO") {
        Cow::Owned(input.replace("TODO", "DONE"))
    } else {
        Cow::Borrowed(input)
    }
}

fn main() {
    let s = good("no changes");
    println!("{s}");
    let s = good("fix TODO");
    println!("{s}");
}
```

---

> 📌 **建议下一步**：使用 `cargo expand` 查看宏与泛型展开结果；配合 `cargo tree` 管理依赖；在生产环境中开启 `RUST_LOG=debug` 与结构化日志（`tracing`）。