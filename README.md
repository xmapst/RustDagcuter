# Dagcuter 🚀

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)  [![Rust](https://img.shields.io/badge/rust-1.70%2B-orange.svg)](https://www.rust-lang.org)  [![Build Status](https://img.shields.io/badge/build-passing-brightgreen.svg)]

[RustDagcuter](https://crates.io/crates/dagcuter) 是一个用于执行任务的有向无环图 (DAG) 的 Rust 库。它管理任务依赖关系，检测循环依赖关系，并支持可自定义的任务生命周期（执行前、执行后）。它还支持并发执行独立任务，以提高性能。

---

## ✨ 核心功能

- **智能依赖管理**：自动解析、调度多任务依赖。
- **循环检测**：实时发现并阻止循环依赖。
- **高并发执行**：拓扑排序驱动并行运行，充分利用多核。
- **指数退避重试**：内置可配置重试策略；支持自定义间隔、倍数及最大次数。
- **优雅取消**：支持中途取消与资源释放。
- **执行追踪**：实时打印任务状态与执行顺序。
- **类型安全**：静态类型保证，编译期错误检查。
- **零成本抽象**：Minimal runtime overhead。
- **生命周期钩子**：任务执行前/后均可插入自定义逻辑。

## 🏗️ 项目结构

```text
dagcuter/
├─ src/
│  ├─ lib.rs            # 核心导出与类型定义
│  ├─ task.rs           # Task 特性与钩子
│  ├─ retry.rs          # 重试策略
│  ├─ cycle_check.rs    # 循环检测算法
│  └─ executor.rs       # 执行器核心
├─ examples/            # 示例代码
├─ Cargo.toml
└─ README.md
````

## 🚀 快速上手

1. 在 `Cargo.toml` 添加依赖：

   ```toml
   [dependencies]
   dagcuter = "0.1.1"
   tokio = { version = "1.0", features = ["full"] }
   async-trait = "0.1"
   serde = { version = "1.0", features = ["derive"] }
   serde_json = "1.0"
   thiserror = "1.0"
   futures = "0.3"
   tokio-util = "0.7"
   ```

2. 编写任务并执行：

   ```rust
   use dagcuter::*;
   use async_trait::async_trait;
   use tokio_util::sync::CancellationToken;
   use std::{collections::HashMap, sync::Arc};

   struct ExampleTask {
       name: String,
       deps: Vec<String>,
   }

   #[async_trait]
   impl Task for ExampleTask {
       fn name(&self) -> &str { &self.name }
       fn dependencies(&self) -> Vec<String> { self.deps.clone() }
       fn retry_policy(&self) -> Option<RetryPolicy> {
           Some(RetryPolicy { max_attempts: 3, ..Default::default() })
       }
       async fn execute(
           &self,
           _ctx: CancellationToken,
           _input: &TaskInput,
       ) -> Result<TaskResult, DagcuterError> {
           println!("执行任务: {}", self.name);
           tokio::time::sleep(std::time::Duration::from_millis(100)).await;
           let mut out = HashMap::new();
           out.insert("status".into(), serde_json::json!("ok"));
           Ok(out)
       }
   }

   #[tokio::main]
   async fn main() {
       let mut tasks: HashMap<String, BoxTask> = HashMap::new();
       tasks.insert("A".into(), Arc::new(ExampleTask { name: "A".into(), deps: vec![] }));
       tasks.insert("B".into(), Arc::new(ExampleTask { name: "B".into(), deps: vec!["A".into()] }));

       let mut engine = Dagcuter::new(tasks).unwrap();
       let ctx = CancellationToken::new();

       println!("=== 依赖图 ===");
       engine.print_graph();

       println!("=== 开始执行 ===");
       let results = engine.execute(ctx.clone()).await.unwrap();
       println!("=== 完成: {:?} ===", results);
   }
   ```

3. 运行示例：

   ```bash
   cargo run
   ```

---

## 📚 API 概览

### `Task` 特性

```rust
#[async_trait]
pub trait Task: Send + Sync {
  fn name(&self) -> &str;
  fn dependencies(&self) -> Vec<String>;
  fn retry_policy(&self) -> Option<RetryPolicy> { None }
  async fn pre_execution(&self, _ctx: CancellationToken, _input: &TaskInput) -> Result<(), DagcuterError> { Ok(()) }
  async fn execute(&self, ctx: CancellationToken, input: &TaskInput) -> Result<TaskResult, DagcuterError>;
  async fn post_execution(&self, _ctx: CancellationToken, _output: &TaskResult) -> Result<(), DagcuterError> { Ok(()) }
}
```

### `RetryPolicy`

```rust
#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct RetryPolicy {
  pub interval: Duration,
  pub max_interval: Duration,
  pub max_attempts: i32,
  pub multiplier: f64,
}

impl Default for RetryPolicy {
  fn default() -> Self { /* 1s, 30s, -1, 2.0 */ }
}
```

### `Dagcuter`

```rust
impl Dagcuter {
  pub fn new(tasks: HashMap<String, BoxTask>) -> Result<Self, DagcuterError>;
  pub async fn execute(&mut self, ctx: CancellationToken) -> Result<HashMap<String, TaskResult>, DagcuterError>;
  pub async fn execution_order(&self) -> Vec<String>;
  pub fn print_graph(&self);
}
```

## 🔧 高级用法

* 自定义重试：调整 `interval`、`multiplier`、`max_attempts`
* 生命周期钩子：重写 `pre_execution`/`post_execution`
* 取消与超时：结合 `CancellationToken` 控制执行
* 复杂数据流：在 `execute` 中处理 `TaskInput` 并返回自定义 `TaskResult`

## 📝 许可证

本项目采用 MIT 协议，详见 [LICENSE](LICENSE)。
