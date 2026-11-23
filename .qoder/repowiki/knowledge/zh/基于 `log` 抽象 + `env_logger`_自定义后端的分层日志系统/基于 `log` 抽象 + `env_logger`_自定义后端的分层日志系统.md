---
kind: logging_system
name: 基于 `log` 抽象 + `env_logger`/自定义后端的分层日志系统
category: logging_system
scope:
    - '**'
source_files:
    - crates/nsh-engine/src/logging.rs
    - crates/nsh-core/src/log_file.rs
    - crates/nsh-cli/src/logger.rs
    - crates/gpui/nsh-platform-gpui/src/dev.rs
    - crates/nsh-js/src/host.rs
    - crates/nsh-platform-apple/src/unified_log.rs
    - Cargo.toml
---

## 1. 使用的框架与工具

仓库使用 Rust 标准日志抽象 [`log`](https://crates.io/crates/log)（workspace `Cargo.toml` 声明 `log = "0.4"`）作为所有 crate 的统一 API，通过 `log::info!` / `log::debug!` / `log::warn!` / `log::error!` 宏输出。具体后端按运行环境拆分：

- **CLI (`nsh-cli`)**：实现 `log::Log` 的自定义 `Logger`（`crates/nsh-cli/src/logger.rs`），将日志渲染为 UI 风格（正常模式仅显示 warn/error、app 自身输出和 `nsh::perf`；verbose 模式显示 level + target）。不依赖 `env_logger`。
- **桌面 release runner（macOS/Linux/Windows 应用）**：`crates/nsh-engine/src/logging.rs` 提供完整的 `install(LogOptions)`，实现 rolling file（`logs/app.log`，5MB 滚动，保留 3 份）、平台输出钩子（统一日志、调试器输出、stderr）以及 panic hook（无论 filter 如何都写入文件）。
- **GPUI 宿主**：`crates/gpui/nsh-platform-gpui/src/dev.rs` 实现 `DevLogger: log::Log`，在开发模式下设置最大级别为 `Info`。
- **示例 app**：模板与示例（`crates/nsh-cli/src/templates/rust/main.rs`、`examples/gallery-rs/native`、`examples/tasks-rs`）调用 `env_logger::init()`，即 `env_logger = "0.11"`。
- **各平台 dev 入口**（Android/iOS/Linux/macOS/Web/Windows）：均通过 `log::set_max_level(LevelFilter::Info)` 控制级别。

顶层还声明了 `env_filter = { version = "2", default-features = false }`，用于解析 `RUST_LOG` 过滤器（注释明确说明“env_logger's filter alone, without regular expressions”）。

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `crates/nsh-engine/src/logging.rs` | 桌面 release runner 的主 logger：filter、rolling file、panic hook、platform output |
| `crates/nsh-core/src/log_file.rs` | 进程级全局路径 `OnceLock<PathBuf>`，记录 logger 写出的文件，供 `element::log_file` / TS `process.logFile` 读取 |
| `crates/nsh-cli/src/logger.rs` | CLI 自定义 `log::Log` 后端，格式化并转发到 UI |
| `crates/gpui/nsh-platform-gpui/src/dev.rs` | GPUI 开发模式的 `DevLogger` |
| `crates/nsh-js/src/host.rs` | JS 运行时中关闭非必要日志（`set_max_level(Error)`） |
| `crates/nsh-platform-apple/src/unified_log.rs` | Apple 平台将日志写入 unified log |
| `Cargo.toml` | 声明 `log`、`env_logger`、`env_filter` 依赖及版本 |

## 3. 架构与约定

### 3.1 两层后端模型

- **`log` facade**：所有业务代码只依赖 `log` crate 的宏，不直接依赖任何后端。
- **后端安装点**：每个可执行入口负责安装一次后端（`log::set_logger` + `log::set_max_level`），且多处用 `if log::set_logger(...).is_ok()` 或 `OnceLock` 保证只装一次。

### 3.2 过滤策略

桌面 runner 的 `Filter` 有两种模式：

1. **默认模式**（未设置 `RUST_LOG`）：仅输出 `Warn`/`Error`，以及 `target == "js"` 的 `Info` 及以上（对应 `console.*`）；development 模式下额外放行 `Info`。
2. **`RUST_LOG` 模式**：使用 `env_filter::Builder` 解析 `RUST_LOG`（如 `warn,my_app=debug,nsh_engine::updates=info`），解析失败回退到默认模式并告警。

CLI 的 `Logger::enabled` 则按 target 区分：`starts_with("nsh")` 或 `== "js"` 全部放行，其他 crate 仅 `<= Warn` 时显示。

### 3.3 结构化字段与目标命名约定

日志不使用结构化字段（无 JSON/KeyedValue），而是以 `target` 字符串划分来源：

- `nsh::build` — bundler 构建警告（`crates/nsh-bundler/src/bundle.rs`、`plugin.rs`）
- `nsh::perf` — 性能监控（canvas/image 绘制耗时，`crates/gpui/nsh-platform-gpui/src/canvas.rs`、`image/turn.rs`）
- `nsh::fps` — FPS 统计（`crates/gpui/nsh-platform-gpui/src/stats.rs`）
- `js` — JS 侧 `console.*` 输出
- 普通模块名（如 `canvas`、`decor`、`agent`）— 各 crate 内部调试信息

CLI 对 `nsh::perf` 有特例：非 verbose 模式下以 `dim("perf")` 前缀单独展示，不被当作普通 info 行吞掉。

### 3.4 日志文件格式

桌面 runner 的文件格式固定为一行一条：

```
<UTC ISO8601 毫秒>.Z <LEVEL> <target>: <message>
```

例如 `2026-11-12T08:30:05.123Z INFO  my_app: signed in`。时间戳由内置的 RFC 3339 实现（`Timestamp`）生成，无需外部 time crate。

### 3.5 Rolling file 与 panic 兜底

- 文件大小超过 `MAX_FILE_BYTES = 5 * 1024 * 1024` 时滚动为 `app.1.log` → `app.2.log` …，最多保留 `KEPT_FILES = 3` 份。
- 文件不可写时等待 `RETRY_AFTER = 30s` 重试，期间丢弃写入但不阻塞。
- panic hook 绕过 filter 强制写入文件，并在 `RUST_BACKTRACE=1` 时附带 backtrace。
- 文件路径通过 `nsh_core::log_file::set` 暴露给上层（`element::log_file` / TS `process.logFile`）。

### 3.6 各平台后端的最小化

各平台 dev 入口统一调用 `log::set_max_level(LevelFilter::Info)`，JS 运行时在 host 初始化时将级别降到 `Error`，避免 JS 层产生大量噪音。

## 4. 约定与约束

- **API 约束**：所有日志通过 `log` crate 的 `info!`/`debug!`/`warn!`/`error!` 宏输出，不直接使用 `println!` 等原始 IO（由 `log::Log` 后端统一接管）。
- **过滤优先级**：`RUST_LOG` 优先于默认规则；解析失败时回退默认规则并打印一次告警（`logging.rs` 第 99–101 行）。
- **目标命名约定**：构建相关日志统一使用 `nsh::build`，性能指标使用 `nsh::perf`，FPS 使用 `nsh::fps`，JS 控制台输出使用 `js`（见 `nsh-cli/src/logger.rs` 与 `nsh-bundler` 源码）。
- **文件位置约定**：桌面 release runner 的日志目录为 `<support_dir>/<app_id>/logs`，主文件名为 `app.log`（`logging.rs` 的 `log_dir` 与 `RollingFile::path` 定义）。
- **并发安全**：rolling file 使用 `Mutex<RollingFile>` 保护，panic 写入使用 `try_lock` + 轮询重试（`PANIC_LOCK_TRIES = 100`），确保 panic 场景下仍能落盘。
- **单实例约束**：`log::set_logger` 只允许设置一次，`nsh-engine::logging::install` 返回 `false` 表示已存在 logger，调用方需处理该情况（`logging.rs` 第 83–103 行）。
- **环境变量**：`RUST_LOG` 控制过滤规则（`env_filter` 语法，不支持正则）；`RUST_BACKTRACE` 控制 panic 是否附带 backtrace。
- **JS 端抑制**：JS 运行时在 `host.rs` 中将 max level 设为 `Error`，避免 JS 层产生 debug/info 日志污染输出。