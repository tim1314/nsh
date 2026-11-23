---
kind: configuration_system
name: NSH 配置系统：nsh.json 能力声明与沙箱化进程环境
category: configuration_system
scope:
    - '**'
source_files:
    - crates/nsh-core/src/access.rs
    - crates/nsh-core/src/default-env
    - crates/nsh-core/src/runtime/process.rs
    - crates/nsh-cli/src/run.rs
    - crates/gpui/nsh-platform-gpui/src/app.rs
    - crates/gpui/nsh-platform-gpui/src/surface.rs
    - examples/counter/nsh.json
    - .github/packages/nsh.json
---

## 1. 使用的系统与总体方法

NSH 没有采用通用的配置库（如 `config`、`dotenv`），而是围绕一个**应用清单文件 `nsh.json`** + **编译期白名单 `default-env`** + **运行时能力检查器 `Access/Capabilities`** 构建的轻量级配置体系。核心思想是“默认拒绝”：所有对外部资源（网络、文件系统、进程、环境变量、宿主 API）的访问都必须显式声明，并由嵌入宿主进一步收紧。

- 清单格式：JSON（`nsh.json`），由 `serde_json` 解析，支持 lenient 模式以兼容未来字段。
- 环境变量：通过 `capabilities.process.env` 白名单控制，结合编译期白名单 `crates/nsh-core/src/default-env`。
- 能力模型：`crates/nsh-core/src/access.rs` 中的 `Capabilities` / `Access` 结构体集中实现所有权限判定。
- 平台差异：各平台后端（`nsh-platform-*`）在启动时把宿主侧的环境变量注入到 JS 层的 `process.env`，但仅允许被白名单命中的键名出现。

## 2. 关键文件与包

- `crates/nsh-core/src/access.rs` — 能力/权限模型的核心实现，包含 `Capabilities`、`Access`、`HostPattern`、`HttpGrant`、`NetworkGrants` 等类型，以及 `allows_env`、`allows_http`、`allows_socket`、`authorize_file` 等判定方法。
- `crates/nsh-core/src/default-env` — 编译进二进制的环境变量白名单文本，列出所有无需额外授权即可读取的元数据变量（`LANG`、`LC_*`、`TZ`、`TERM`、`XDG_CURRENT_DESKTOP` 等）。
- `crates/nsh-core/src/runtime/process.rs` — JS 层 `process.env` 的实现，按白名单过滤宿主环境变量后暴露给应用。
- `examples/*/nsh.json` — 示例应用的清单，典型字段包括 `name`、`id`、`version`、`description`、`homepage`、`license`、`architecture`、`extract_dir`、`bin` 等。
- `.github/packages/nsh.json` — NSH CLI 自身的发布清单模板，使用 `${NSH_VERSION}`、`${NSH_HASH}` 占位符供 CI 替换。
- `crates/nsh-cli/src/run.rs` — CLI 在运行各平台应用时，根据 `capabilities.process.env` 把宿主环境变量映射为命令行参数（iOS `xcrun simctl launch`、Android `adb shell am start` 等）。
- `crates/gpui/nsh-platform-gpui/src/app.rs`、`surface.rs`、`chrome.rs` — GPUI 宿主中把宿主环境注入到 JS 运行时并处理 `nsh.json` 的 `window`、`theme`、`appearance` 等字段。

## 3. 架构与设计约定

### 3.1 清单文件 `nsh.json`

每个 NSH 应用根目录放置 `nsh.json`，描述应用身份、版本、元数据和 `capabilities`。文档注释中给出了完整的能力 JSON 示例（`network`、`webview`、`statusBar`、`application`、`homeDir`、`host`、`openUrl`）。对于无声明的“独立应用”，框架保留一组宽松默认权限；对打包/嵌入的应用，则必须显式声明所需能力。

### 3.2 环境变量白名单机制

`access.rs::allows_env` 的实现逻辑：
1. 从编译期嵌入的 `default-env` 文件中逐行读取，忽略 `#` 注释和空行。
2. 如果请求的变量名与任一白名单名称匹配（不区分大小写），直接放行。
3. 否则检查 `capabilities.process.env` 列表中是否有匹配的条目。
4. 两者都不匹配 → 该变量对应用不可见。

`default-env` 的设计约束（由文件头注释和测试共同保证）：
- 只包含“呈现元数据”（语言、时区、终端能力、桌面会话类型），不包含用户身份、路径、代理地址或会话地址。
- 每个变量名独占一行，大写形式，`#` 起注释。
- 测试 `default_environment_entries_are_upper_case_names` 强制每个条目只含 ASCII 大写字母、数字和下划线。

### 3.3 运行时 `process.env` 过滤

JS 层的 `process.env` 不是直接暴露宿主环境，而是经过两层过滤：
- 第一层：`Access::allows_env` 判断变量名是否被白名单或 `capabilities.process.env` 授予。
- 第二层：`runtime/process.rs` 中 `get` 方法仅在变量名同时出现在宿主实际环境和白名单列表时才返回值。

CLI 在启动模拟器/设备时，会把 `capabilities.process.env` 中声明的变量作为命令行参数传入（iOS 用 `xcrun simctl launch`，Android 用 `adb shell am start`），确保目标进程确实能读到这些变量。

### 3.4 能力模型的“双重许可”

`Access` 由 `declared`（应用声明）和 `granted`（宿主授予）两部分组成，任何能力必须**两者都允许**才生效。这使宿主可以进一步收紧应用权限，例如禁止某个 host 函数调用或网络主机。

### 3.5 Lenient 解析

`Capabilities::from_json_lenient` 允许未知字段存在但不授予其能力，并在工具链层面发出警告，从而保证向后兼容的同时仍能在构建期发现拼写错误。

## 4. 约定与约束

- **默认拒绝原则**：除明确列出的能力外，所有外部访问均被拒绝。来源：`access.rs` 模块文档注释及 `denies_unless_listed` 测试。
- **环境变量白名单**：只有 `default-env` 中列出的变量名和应用声明的 `capabilities.process.env` 中的名称才会出现在 `process.env`。来源：`allows_env` 实现及 `default_environment_entries_are_upper_case_names` 测试。
- **环境变量命名规范**：`default-env` 中的条目必须是纯 ASCII 大写字母、数字和下划线的组合。来源：`access.rs` 中的单元测试。
- **敏感变量排除**：`USER`、`HOME`、`HOSTNAME`、`PATH`、`TMPDIR`、`XDG_*_HOME`、`DISPLAY`、`DBUS_SESSION_BUS_ADDRESS`、`SSH_AUTH_SOCK`、`HTTP_PROXY*` 等敏感变量不在白名单中，应用需显式申请。来源：`default-env` 文件头注释。
- **能力声明可 lenient 解析**：新版本声明的未知字段不会导致解析失败，但不会被授予能力，且工具链会告警。来源：`access.rs` 模块文档注释。
- **`nsh.json` 是应用清单而非通用配置中心**：仓库中没有统一的配置文件加载器；各示例应用各自维护自己的 `nsh.json`，CLI 和宿主在启动阶段读取它。
- **CLI 自身配置通过 Cargo 环境变量注入**：`crates/nsh-cli/build.rs` 使用 `SOURCE_DATE_EPOCH`、`CARGO_CFG_TARGET_OS`、`CARGO_CFG_TARGET_ENV` 等构建期环境变量生成 CLI 元信息。
- **发布清单模板使用占位符**：`.github/packages/nsh.json` 中的 `${NSH_VERSION}`、`${NSH_REPO}`、`${NSH_HASH}` 由 CI 替换。来源：文件内容本身。

## 5. 未覆盖的范围

- 未发现通用的 `.env` 文件加载、YAML/TOML 配置中心、Feature Flag 系统或密钥管理方案。
- 主题/样式配置（`theme.json`、`appearance`）属于 UI 层，不属于本分类讨论的运行时配置系统。
- 多平台宿主各自的启动参数（如 macOS `Info.plist`、Android manifest）不在本分析范围内。