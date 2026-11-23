---
kind: error_handling
name: NSH 错误处理体系：JsError、PropError 与领域错误类型
category: error_handling
scope:
    - '**'
source_files:
    - crates/nsh-core/src/element/system.rs
    - crates/nsh-core/src/element/value.rs
    - crates/nsh-core/src/host_channel.rs
    - crates/nsh-core/src/main_thread.rs
    - crates/nsh-core/src/color.rs
    - crates/nsh-core/src/desktop.rs
    - crates/nsh-core/src/desktop/login_item.rs
    - crates/nsh-core/src/audio.rs
    - crates/nsh-core/src/mobile.rs
    - crates/nsh-core/src/js/ops.rs
    - crates/nsh-core/src/element/component.rs
    - crates/nsh-base/src/theme.rs
    - crates/nsh-base/src/rich_editor/extensions/extension.rs
    - crates/nsh-base/src/rich_editor/formats/mod.rs
    - crates/nsh-bundler/src/bundle.rs
    - crates/nsh-cli/src/ui.rs
---

## 1. 总体方案

NSH 是一个 Rust + TypeScript/JSX 的跨平台原生 Shell 框架，错误处理以「Rust 侧定义结构化错误类型 → 映射到 JS 侧 `Error` / `TypeError` / `RangeError`」为主线。核心库 `nsh-core` 暴露统一的 `JsError` 作为 JS 层抛出的错误；各子系统（host channel、main thread、color、desktop、audio、mobile、rich editor、bundler、CLI）各自定义领域错误枚举，并通过 `From` 或显式转换落到 `JsError`。

没有发现全局的 panic/recover 中间件；panic 仅在少数基础设施路径被捕获并转为错误（如 `MainThreadError::Panicked`），其余代码使用 `Result<T, E>` 和 `?` 传播。

## 2. 关键文件与类型

- **`crates/nsh-core/src/element/system.rs`** — 定义 `JsError`、`ErrorClass`、`ErrorFields` 以及 Node.js 风格的错误码常量模块 `codes`（`ACCESS_DENIED`、`UNAVAILABLE`、`QUOTA_EXCEEDED`、`TIMED_OUT`、`HOST_ERROR`、`WEBVIEW_SCRIPT`、`WEBVIEW_ABORTED`、`WEBVIEW_TERMINATED`、`DOWNLOAD_*` 等）。提供 `JsError::with_code`、`JsError::access_denied`、`io_code` 等构造器，并将 `std::io::Error` 转换为带 OS 错误码的 `JsError`。
- **`crates/nsh-core/src/element/value.rs`** — 定义 `PropError(pub String)`，用于 JS→Rust 属性解析失败（`JsValue::from_js` 返回 `Result<Self, PropError>`），并提供 `PropError::expected(ty, value)` 生成 "expected X, got Y" 消息。
- **`crates/nsh-core/src/host_channel.rs`** — 定义 `HostError`（`Failed`、`Denied`、`InvalidName`、`Unavailable`、`TimedOut`、`TooManyCalls`），实现 `code()` 映射到稳定字符串，并通过 `impl From<HostError> for JsError` 统一转成 JS 可消费的错误。
- **`crates/nsh-core/src/main_thread.rs`** — 定义 `MainThreadError`（`Panicked(String)`、`Dropped`），在 `task` 中用 `std::panic::catch_unwind` 捕获 panic 并转为 `Err(MainThreadError::Panicked(...))`，由 `OnMainThread` Future 返回。
- **`crates/nsh-core/src/color.rs`** — `ColorParseError(pub String)`，实现 `std::error::Error`。
- **`crates/nsh-core/src/desktop.rs`** — `ShortcutError(pub String)`。
- **`crates/nsh-core/src/desktop/login_item.rs`** — `LaunchAtLoginError`，实现 `source()` 链式溯源。
- **`crates/nsh-core/src/audio.rs`** — `AudioError` 枚举。
- **`crates/nsh-core/src/mobile.rs`** — `AppSettingsError` 枚举。
- **`crates/nsh-core/src/js/ops.rs`** — `OpsError` 枚举。
- **`crates/nsh-core/src/element/component.rs`** — `ComponentError`。
- **`crates/nsh-base/src/theme.rs`** — `ThemeError(pub String)`。
- **`crates/nsh-base/src/rich_editor/** 下多个子模块分别定义 `ExtensionError`、`KeymapError`、`FormatError`、`ReplaceError`、`PosError`、`SchemaError`、`StateError`、`StepError`，均实现 `std::error::Error`。
- **`crates/nsh-bundler/src/bundle.rs`** — `BundleError`，包装 rolldown 诊断。
- **`crates/nsh-cli/src/ui.rs`** — `Failure`，`clap::error::ErrorKind::DisplayVersion` 分支用于版本显示。

## 3. 架构与约定

### 3.1 JS 层错误模型 (`JsError`)
`JsError` 是 NSH 面向 JS 的唯一错误类型，分为三类：
- `Type(String)` → JS `TypeError`
- `Range(String)` → JS `RangeError`
- `Error(String)` → JS `Error`
- `Fields(Box<ErrorFields>)` → 带额外属性的错误，通过 `with(key, value)` 和 `with_code(code)` 追加字段，模拟 Node.js 风格（`code`、`syscall`、`path`、`errno` 等）。

`ErrorClass` 枚举与 JS 内置异常类一一对应，`ErrorFields` 维护有序字段列表，`field(&key)` 支持按名查询。

### 3.2 权限与能力错误
`JsError::access_denied(message, permission, resource)` 统一产出 `ERR_ACCESS_DENIED` 错误，附带 `permission`（`"fs"`、`"network"`、`"clipboard"` 等）和 `resource` 字段，对应 Node.js 权限模型。`HostError::Denied` 也映射为同一 code。

### 3.3 I/O 错误标准化
`impl From<std::io::Error> for JsError` 调用 `io_code`，在 Unix 上尝试 `raw_os_error()` 映射到 errno 名称（`ENOENT`、`EACCES` 等），否则回退到 `ErrorKind` 匹配表。所有 I/O 错误最终都带上稳定的 `code`。

### 3.4 属性解析错误 (`PropError`)
JS→Rust 的属性值转换通过 `JsValue::from_js` 返回 `Result<T, PropError>`。`PropError::expected("a boolean", value)` 会截断过长的 value 到 40 字符并追加 `…`，保证错误消息可读。

### 3.5 主线程任务异常
`main_thread::task` 用 `std::panic::catch_unwind` 包裹用户函数，panic 信息经 `crate::element::panic_message(payload)` 提取后放入 `MainThreadError::Panicked`，Future 端收到 `Err` 而非崩溃。

### 3.6 宿主通道错误 (`HostError`)
`HostError` 集中表达 host call/post 的所有失败原因，并通过 `impl From<HostError> for JsError` 统一转成 JS 错误。`code()` 方法为每个变体返回稳定的 TS 可分支的错误码。

## 4. 约定与约束

- **领域错误类型命名**：各模块定义 `<Domain>Error` 枚举/结构体（`HostError`、`MainThreadError`、`PropError`、`ThemeError`、`AuthSessionError`、`ColorParseError`、`ShortcutError`、`LaunchAtLoginError`、`AudioError`、`AppSettingsError`、`OpsError`、`ComponentError`、`ExtensionError`、`KeymapError`、`FormatError`、`ReplaceError`、`PosError`、`SchemaError`、`StateError`、`StepError`、`BundleError`、`DevError`、`BuildError`、`Restart`、`Failure`），并实现 `std::error::Error`。
- **JS 错误必须走 `JsError`**：面向 JS 层的错误统一收敛到 `JsError`，通过 `From` 实现自动转换（如 `HostError`、`std::io::Error`），避免直接抛出裸 `String`。
- **错误码集中管理**：业务错误码集中在 `system.rs` 的 `codes` 模块（`ACCESS_DENIED`、`UNAVAILABLE`、`QUOTA_EXCEEDED`、`TIMED_OUT`、`HOST_ERROR`、`WEBVIEW_*`、`DOWNLOAD_*`），TS 侧据此分支处理。
- **权限拒绝统一语义**：能力不足一律使用 `JsError::access_denied` 或 `HostError::Denied`，保持 `ERR_ACCESS_DENIED` + `permission` + `resource` 三元组。
- **属性解析错误使用 `PropError::expected`**：类型不匹配时统一用该构造器生成 "expected X, got Y" 消息，便于前端展示。
- **panic 仅用于不可恢复的内部状态**：生产代码中 panic 极少出现，主要出现在测试、示例和极少数基础设施路径（如 `main_thread::task` 捕获 panic 后转为错误）。
- **`#[non_exhaustive]` 标记扩展点**：`HostError`、`MainThreadError` 等对外暴露的错误枚举加 `#[non_exhaustive]`，允许未来新增变体而不破坏下游 ABI。
- **错误消息面向人**：注释明确说明 "messages are for people and may change"，而 `code` 才是 API 契约。

## 5. 未观察到的模式

- 未发现全局 panic/recover 中间件或统一日志上报机制。
- 未发现仓库级统一的错误码枚举（除 `JsError::codes` 外，其他错误多为本地枚举）。
- 未发现 `anyhow`/`thiserror` 的使用——错误类型均为手写结构体/枚举 + `std::error::Error` 实现。
- `unwrap()`/`expect()` 大量存在于测试和 GPUI 后端内部，但不在 NSH 公共 API 边界处使用。