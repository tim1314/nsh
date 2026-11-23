---
kind: dependency_management
name: Cargo Workspace 多包依赖管理与版本锁定
category: dependency_management
scope:
    - '**'
source_files:
    - Cargo.toml
    - Cargo.lock
    - .cargo/config.toml
    - rust-toolchain.toml
    - website/package.json
---

## 1. 使用的系统/方法

仓库采用 **Cargo workspace**（resolver "3"）作为 Rust 侧的依赖管理核心，配合 `Cargo.lock` 进行全仓依赖锁定；网站子项目使用 **Bun + npm**（`website/package.json`）管理 Astro、Tailwind、CodeMirror 等前端依赖。Rust 工具链通过 `rust-toolchain.toml` 固定为 `stable`，并显式声明了 Apple iOS/iOS-Sim、Android (aarch64/armv7/x86_64) 等多平台 target。

## 2. 关键文件

- `Cargo.toml`：workspace 根定义、`[workspace.dependencies]` 集中声明所有第三方 crate 与内部 crate 的版本，`default-members = ["crates/xtask"]`，`publish = false`（整个 workspace 不发布到 crates.io）。
- `Cargo.lock`：全仓锁定的依赖树快照。
- `.cargo/config.toml`：设置 `IPHONEOS_DEPLOYMENT_TARGET=15.0`、`xtask` alias、wasm32 目标栈大小与 getrandom backend cfg。
- `rust-toolchain.toml`：固定 toolchain channel 与跨平台 targets。
- `website/package.json`：Astro 站点的前端依赖，通过 Bun 脚本运行。
- 各 `crates/*/Cargo.toml`：引用 workspace 中定义的依赖（如 `nsh-core = { version = "0.6.0", path = "crates/nsh-core" }`），并通过 `default-features = false` + 按需启用 feature 控制编译产物体积。

## 3. 架构与约定

- **集中式依赖声明**：所有第三方 crate 统一在根 `Cargo.toml` 的 `[workspace.dependencies]` 中声明，子 crate 通过名称引用而不重复写版本号，保证全仓一致。
- **内部 crate 以 path + version 双重绑定**：如 `nsh-core = { version = "0.6.0", path = "crates/nsh-core" }`，同时约束语义化版本与本地路径。
- **Feature-gated 依赖**：大量 crate 使用 `default-features = false` 并在需要时开启具体 feature（如 `resvg`、`wgpu`、`tokio`、`objc2-*` 系列），避免默认拉入不必要的框架头与 dylib，减少编译时间与启动开销。
- **精确锁定关键依赖**：对易变或强耦合的依赖使用 `=` 精确版本，例如 `rquickjs = { package = "quickjs-jit", version = "=0.14.2" }`、`rolldown = "=1.2.13"`、`string_wizard = "=1.2.13"`、`oxc = "=0.153.0"`、`quickjs-jit-stdlib = "=0.14.2"`、`ureq = "~3.5.2"`。
- **按包优化 profile**：根 `Cargo.toml` 中对 `quickjs-jit-sys`、`taffy`、`syntect`、`fancy-regex`、`onig_sys`、`two-face`、`tiny-skia`、`resvg`、`image` 等热路径 crate 在 dev profile 下单独设置 `opt-level = 3`，加速开发体验。
- **Web 专属 profile**：`[profile.web]` 继承 release 并将 `opt-level = "z"`、`strip = true`、`panic = "abort"`，针对 `nsh.wasm` 体积优化。
- **GPUI 子 workspace 隔离**：根 workspace 通过 `exclude = ["crates/gpui"]` 排除 GPUI 后端，其自身是独立 Cargo workspace（有独立的 `crates/gpui/Cargo.toml` 与 `crates/gpui/Cargo.lock`）。
- **Apple 构建环境变量**：`.cargo/config.toml` 中 `IPHONEOS_DEPLOYMENT_TARGET = "15.0"` 与 `nsh-cli/src/apple.rs` 中的 `IOS_MINIMUM` 保持一致，确保打包产物最低系统版本统一。
- **WASM 目标特殊配置**：为 `wasm32-unknown-unknown` 设置 `-C link-arg=-zstack-size=8388608`（8 MiB）和 `getrandom_backend="custom"`，以满足 QuickJS 递归与组件树的栈需求。

## 4. 约定与约束

- **Workspace 内所有 Rust 依赖必须集中在根 `[workspace.dependencies]` 声明**：子 crate 仅通过名称引用，禁止在子 crate 中重复声明同一 crate 的不同版本（由 workspace resolver 强制）。
- **内部 crate 版本与 workspace.package.version 同步**：所有 `nsh-*` 内部 crate 在 workspace.dependencies 中统一声明 `version = "0.6.0"`，新增 crate 需遵循此模式。
- **外部依赖优先使用语义化版本范围**：如 `anyhow = "1"`、`clap = { version = "4.6", features = ["derive"] }`，仅在存在兼容性风险时使用 `=` 精确锁定（见 rquickjs、rolldown、oxc 等）。
- **Feature 最小化原则**：对大型 crate（wgpu、resvg、objc2-*、tokio、gtk4 等）一律关闭 default-features，仅启用实际用到的功能模块，以减少链接体积与编译时间。
- **多平台 target 需在 `rust-toolchain.toml` 中显式注册**：当前包含 aarch64-apple-darwin、aarch64-apple-ios(-sim)、aarch64/armv7/x86_64-linux-android。
- **网站前端依赖通过 Bun 管理**：`website/package.json` 使用 `bunx --bun astro` 等脚本，未出现 `package-lock.json` / `yarn.lock`，依赖解析由 Bun 处理。
- **Workspace 整体不发布**：根 `Cargo.toml` 中 `publish = false`，只有被显式发布的 crate（如有）才会上架 crates.io。
- **CI/构建脚本通过 xtask 统一入口**：`.cargo/config.toml` 定义 `alias.xtask = "run -q -p xtask --"`，所有工作区级任务经 `crates/xtask` 执行。