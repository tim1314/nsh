---
kind: build_system
name: NSH 构建系统：Cargo workspace + xtask + GitHub Actions 多平台发布流水线
category: build_system
scope:
    - '**'
source_files:
    - Cargo.toml
    - rust-toolchain.toml
    - crates/xtask/Cargo.toml
    - .github/workflows/release.yml
    - .github/workflows/packages.yml
    - .github/workflows/core.yml
    - .github/workflows/linux.yml
    - .github/workflows/windows.yml
    - .github/workflows/gpui.yml
    - .github/workflows/examples.yml
    - .github/workflows/ci-paths.yml
    - .github/workflows/changed.yml
---

## 1. 总体方案

仓库是一个 Rust Cargo workspace（`resolver = "3"`，`edition = "2024"`，`rust-version = "1.97"`），所有 crate 统一在根 `Cargo.toml` 的 `[workspace]`、`[workspace.package]`、`[workspace.dependencies]`、`[workspace.lints.rust]` 中声明版本与 lint 规则。构建入口是 `crates/xtask`，被设为 `default-members`；`cargo run` 默认运行 gallery，`cargo xtask --help` 暴露维护者任务（gallery、Android 构建、发行打包）。

跨平台分发由 `.github/workflows/release.yml` 驱动，调用 `cargo --locked xtask dist -v`，在 macOS、Linux、Windows 三个矩阵上并行构建 CLI、桌面运行时和嵌入 SDK，产物上传为 GitHub Release 资产，并触发 Homebrew tap 更新。

CI 由多个独立的 workflow 组成：`core.yml`（核心库）、`linux.yml`（GTK 后端 + 示例测试）、`windows.yml`（Win32 后端）、`gpui.yml`（独立 workspace `crates/gpui`）、`examples.yml`（遍历 examples 下所有 TSX/Rust 应用）、`packages.yml`（Homebrew formula 生成与推送）、`compat.yml`、`release-crate.yml`、`tui.yml`、`website.yml`、`changed.yml`、`ci-paths.yml`。

## 2. 关键文件

- `Cargo.toml` — workspace 定义、共享依赖、`profile.release`（`opt-level=3, lto="fat", codegen-units=1`）、`profile.web`（继承 release，`opt-level="z"`, `strip=true`, `panic="abort"`）、对 quickjs/taffy/syntect/tiny-skia/image 等热路径的 `profile.dev.package.*.opt-level=3`。
- `rust-toolchain.toml` — 固定 `channel = "stable"`，预注册 iOS / Android / Mac Catalyst 目标。
- `crates/xtask/Cargo.toml` — 维护者任务 crate，描述为 `Maintainer tasks: cargo run (the gallery), Android builds and the distribution`。
- `.github/workflows/release.yml` — 发行主流程：tag `vX.Y.Z` 触发全平台构建 → draft release；nightly cron（UTC 18:30，新加坡时间 02:30，周三跳过）→ 移动 `nightly` tag；`workflow_dispatch` 支持手动指定 tag 或 nightly。
- `.github/workflows/packages.yml` — 稳定版发布后自动生成 Homebrew formula，校验 SHA256，安装并 `brew test`，然后 push 到 `longbridge/homebrew-tap`。
- `.github/workflows/core.yml` / `linux.yml` / `windows.yml` / `gpui.yml` / `examples.yml` — 各平台 CI，均通过 `paths:` 精确限定触发范围。
- `.github/workflows/ci-paths.yml` — PR 必跑，用 Python 脚本校验每个 workflow 的 `paths` 列表与其实际依赖图一致，防止漏测。
- `.github/actions/cargo/` — 自定义 Action，封装 huacnlee/cargo 缓存（key 按 `linux`/`windows`/`core`/`gpui`/`examples` 区分）。
- `website/public/install.sh` / `website/public/install.ps1` — 随 release 附带的安装器，下载无版本号命名的资产（如 `nsh-macos-arm64.tar.gz`）。

## 3. 架构与约定

### 3.1 版本策略
- 单一来源：`Cargo.toml` 的 `[workspace.package].version = "0.6.0"`，所有 crate 使用 `version.workspace = true`。
- 发行时强制校验：`release.yml` 在 `TAG != ''` 时执行 `sed` 提取 `Cargo.toml` 中的 version，要求 `TAG == v$version`，否则退出。
- Nightly 与 stable 共用同一 workspace 版本，但通过 `NSH_CHANNEL=nightly` 环境变量区分，CLI 内置 `nsh update nightly/stable` 切换。

### 3.2 构建产物命名
- 资产名不带版本号：`nsh-macos-arm64.tar.gz`、`nsh-macos-x86_64.tar.gz`、`nsh-linux-x86_64.tar.gz`、`nsh-linux-arm64.tar.gz`、`nsh-windows-x86_64.zip`、`nsh-windows-arm64.zip`，以及各自的 `.sha256`。
- 每个归档内含 `nsh/` 目录，布局同 `dist/nsh`：包含该平台的 CLI 与运行时，以及 `nsh-android.tar.gz`、`nsh-web.tar.gz`、`nsh-android-symbols.tar.gz`、`nsh-ios-macos-arm64.tar.gz`、`nsh-gpui-<platform>.tar.gz`、`nsh-tui-<platform>.tar.gz` 等组件，由已安装的 `nsh component` 按需下载。
- 嵌入 SDK 单独产出 `nsh-embed-ios.zip`、`nsh-embed-android.zip`。

### 3.3 平台矩阵
- `release.yml` 的 matrix 包含 `macos-arm64`（Xcode 26+，iOS SDK iphoneos/iphonesimulator/macosx 均需 ≥26）、`macos-x86_64`、`linux-x86_64`（Ubuntu 24.04，GTK 4.14+，glibc 2.39+）、`linux-arm64`、`windows-x86_64`、`windows-arm64`。
- Linux 构建额外安装 clang、GTK 4、WebKitGTK 6.0、GStreamer、Wayland、X11、xkbcommon、fontconfig、FreeType。
- Windows 设置 `LIBCLANG_PATH=C:\Program Files\LLVM\bin` 与 `RUST_MIN_STACK=268435456` 避免 QuickJS bindgen 与 `windows` crate 栈溢出。
- WebAssembly C 工具链通过 `CC_wasm32_unknown_unknown` / `AR_wasm32_unknown_unknown` 注入 LLVM clang/llvm-ar，Windows 上通过 Python wrapper 规范化 `\\?\` 路径。

### 3.4 优化配置
- 发行 LTO：默认 `fat`，但 release workflow 覆盖为 `CARGO_PROFILE_RELEASE_LTO=thin` + `CODEGEN_UNITS=16`，注释说明 fat LTO 单线程链接 ~15 个可执行体需 5–25 分钟/个，thin LTO 耗时少得多且运行时成本几乎不变。
- Web profile 保持 `fat` LTO + `codegen-units=1`，因为 `nsh.wasm` 是页面下载的产物，优先体积。
- CI 禁用增量编译与 debug info：`CARGO_INCREMENTAL=0`、`CARGO_PROFILE_DEV_DEBUG=0`。

### 3.5 CI 调度与去抖
- 所有 workflow 对 schedule 事件采用 `cron: "0 */3 * * *"`（每 3 小时），而非每次 push 都跑，以节省缓存。
- 每个 workflow 首 job 调用 `.github/workflows/changed.yml`，仅当 main 自上次运行以来有变动才继续。
- 通过 `environment.name: ci-debounce` + `deployment: false` 实现 debounce，使较新的 push 取消排队中的旧运行。
- PR 关闭或草稿 PR 不运行检查。

### 3.6 示例与测试覆盖
- `examples.yml` 动态发现所有含 `mount(` 的 `.tsx` 入口与含 `"root"` 的 `nsh.json`，逐一 `nsh build`。
- `linux.yml` 与 `windows.yml` 分别通过 `xvfb-run` / Win32 窗口运行 `nsh test --target linux/windows` 下的 gallery 测试。
- `gpui.yml` 在三个 OS 上构建独立 workspace `crates/gpui`，并在 Linux 上以 `--features shot` 运行截图测试。
- `core.yml` 额外 `cargo check -p nsh-core -p nsh-base --all-features --target wasm32-unknown-unknown` 确保 web 侧 cfg 正确。

## 4. 约定与约束

- **版本必须与 tag 匹配**：`release.yml` 在带 tag 的发行中解析 `Cargo.toml` 的 version，若 `TAG != v$version` 直接失败（见第 183–190 行）。
- **发行产物不可变**：所有资产名不含版本号，`releases/latest/download/<asset>` 始终指向最新发行；SHA256 摘要随资产一起发布，`packages.yml` 在安装前验证两个二进制资产的 checksum。
- **Nightly 每周三跳过**：`release.yml` 计划任务在新加坡时间周三（Singapore timezone `%u == 3`）跳过构建，让稳定版发布刷新 nightly，保证 `install.sh --nightly` 不低于 stable。
- **iOS SDK 最低版本 26**：`release.yml` 在 `matrix.embed == 'ios'` 时检查 `iphoneos`、`iphonesimulator`、`macosx` 三个 SDK 的主版本号，低于 26 报错。
- **Linux 运行时依赖 GTK 4.14+、Pango 1.52+、glibc 2.39+**：由 `release.yml` 注释与 Ubuntu 24.04 runner 保证。
- **CI paths 必须与依赖图同步**：`.github/workflows/ci-paths.yml` 在每个 PR 上运行 `.github/ci/paths.py`，校验各 workflow 的 `paths` 列表是否覆盖了其实际依赖的 crate，未命中则失败。
- **Web 构建走专用 profile**：`Cargo.toml` 定义 `[profile.web]` 继承 release 并将 `opt-level` 改为 `"z"`、启用 `strip` 与 `panic = "abort"`，由 website 与 bundler 消费。
- **默认成员为 xtask**：`default-members = ["crates/xtask"]`，仓库级 `cargo run` 等价于 `cargo run -p xtask`，即启动 gallery。
- **GPUI 是独立 workspace**：根 `Cargo.toml` 显式 `exclude = ["crates/gpui"]`，由 `crates/gpui/Cargo.toml` 作为自己的 workspace 管理，CI 通过 `gpui.yml` 单独构建。
- **发布 Homebrew formula 需要 secret**：`packages.yml` 的 `homebrew` job 要求 `HOMEBREW_TAP_TOKEN` 存在且具有 `longbridge/homebrew-tap` 的 Contents write 权限，否则直接报错退出。
- **Self-hosted runner 缓存策略**：release workflow 对 `runner.environment == 'self-hosted'` 使用 `.github/actions/cargo` Action，利用 AGENTS.md 描述的 `huacnlee/cargo` 外部 build dir，使 checkout clean 后仍能增量构建。