---
kind: frontend_style
name: NSH 前端样式体系：Rust 原生 Tailwind 解析器 + OKLCH 设计令牌
category: frontend_style
scope:
    - '**'
source_files:
    - crates/nsh-core/src/tailwind/mod.rs
    - crates/nsh-core/src/tailwind/utilities.rs
    - crates/nsh-core/src/tailwind/theme.rs
    - crates/nsh-core/src/tailwind/palette.rs
    - crates/nsh-core/src/style.rs
    - crates/nsh-base/src/default-theme.json
    - website/src/dark.theme.json
    - website/src/light.theme.json
    - website/package.json
---

## 1. 采用的系统/方法

NSH 的 UI 样式体系由两层组成：

- **运行时样式引擎（核心）**：在 `crates/nsh-core/src/tailwind/` 中用 Rust 实现了一个 **Tailwind CSS v4 类的解析与求值器**。JS 侧只传递 class 字符串，Rust 端将其解析为规则、缓存到 LRU（上限 4096），再根据 appearance/window size/state/group/data-* 等上下文求值为 NSH 内部的 `Style` / `ClassProps`。它不是把 CSS 交给浏览器渲染，而是直接映射到布局、颜色、阴影、变换等原生属性。
- **主题/设计令牌**：使用 JSON 文件定义色板与 token，默认主题位于 `crates/nsh-base/src/default-theme.json`，网站自身主题位于 `website/src/dark.theme.json` 与 `website/src/light.theme.json`。颜色统一使用 OKLCH 空间（如 `oklch(0.205 0 0)`），并在 theme 层转换为 sRGB。
- **文档站点（Astro）**：`website/package.json` 依赖 `tailwindcss ^4.1.15` 与 `@tailwindcss/vite ^4.1.15`，通过 Vite 插件编译；站点本身是 Astro 静态站，不依赖 React 组件库，而是用 Astro 页面 + Shiki 代码高亮。

## 2. 关键文件

- `crates/nsh-core/src/tailwind/mod.rs` — Tailwind 类解析入口、`ClassList`/`Rule`/`Rank`/`Condition` 模型、状态位（`state::*`）、环境位（`deps::*`）、平台变体（`Platform::Ios|Android|Web`）、指针精度（`Pointer::Coarse|Fine`）以及 `Tailwind` 缓存。
- `crates/nsh-core/src/tailwind/utilities.rs` — 具体 utility 类（间距、尺寸、颜色、字体、阴影、变换等）的解析与声明生成。
- `crates/nsh-core/src/tailwind/theme.rs` — 从 `Tokens` 构建 Tailwind 主题、token 比较与共享逻辑。
- `crates/nsh-core/src/tailwind/palette.rs` — OKLCH → sRGB 调色板转换。
- `crates/nsh-core/src/style.rs` — 底层 `Style`、`Color`、`FontFamily`、`Effects`（阴影、渐变、过渡、动画）等样式数据结构。
- `crates/nsh-base/src/default-theme.json` — 内置默认主题（light/dark 两套 oklch 色值，含 `primary`/`destructive`/`chart-*`/`sidebar-*` 等语义 token）。
- `website/src/dark.theme.json` / `website/src/light.theme.json` — Astro 站点的 Shiki 语法高亮主题。
- `website/package.json` — 声明 Tailwind v4 与 Astro 依赖。

## 3. 架构与约定

### 3.1 Tailwind 类 → 原生样式的映射

`mod.rs` 的模块注释明确描述了优先级链：**默认样式 → class 规则 → inline `style`（`Op::SetStyle`）**；带 `!` 的 utility 会覆盖 inline style。每个 class 串被解析一次并缓存，LRU 上限 4096，避免动态生成的 class 无限增长。

状态与条件通过位掩码表达：
- `state::*`：`PRESSED`/`HOVERED`/`FOCUSED`/`DISABLED`/`FOCUS_WITHIN`/`SELECTED`/`CHECKED`/`INDETERMINATE`/`EXPANDED`/`ARIA_PRESSED`/`BUSY`/`INVALID`/`FOCUS_VISIBLE`。
- `position::*`：`FIRST`/`LAST`/`ODD`/`EVEN`/`ONLY`。
- `deps::*`：`DARK`/`WIDTH`/`HEIGHT`/`RTL`/`STATE`/`GROUP`/`POSITION`/`MOTION`/`THEME`/`CURRENT_COLOR`/`CH`/`DATA`/`DEVICE`，用于决定何时重新求值。

变体排序遵循 Tailwind 的注册顺序：plain → group → pseudo-elements (`placeholder:`/`selection:`) → position (`first:`/`odd:`…) → interaction (`checked:`/`selected:`/`focus-within:`/`hover:`/`focus:`/`focus-visible:`/`active:`/`enabled:`/`disabled:`) → aria-* → data-* → motion-safe/reduce → breakpoint (`max-*` 降序, `min-*` 升序) → portrait/landscape → LTR/RTL → dark → pointer-fine/coarse → platform (`ios:`/`android:`/`web:`) → terminal。

### 3.2 主题与 Token

默认主题采用 shadcn/ui 风格的命名空间：`background`/`foreground`/`card`/`popover`/`primary`/`secondary`/`muted`/`accent`/`destructive`/`border`/`input`/`ring`/`chart-1..5`/`sidebar*`，分 `light` 与 `dark` 两个子对象，全部使用 OKLCH 表示。`default-theme.json` 还暴露顶层 `cssVars.theme.radius`。

应用可通过 `nsh.json` 中的 `tailwind` 字段扩展 Tailwind 主题；引擎启动时把 `nsh:tailwind-theme` 作为 root prop 注入 JS 侧。

### 3.3 响应式与多平台策略

- 断点基于窗口宽度（`w-screen`、`min-*`/`max-*` 单位支持 `pt`/`px`/`rem`）。
- 方向感知：`rtl:` 与 `ltr:` 变体，配合 `space-x-reverse`/`space-y-reverse` 处理 RTL 布局。
- 设备能力：`pointer-fine:`/`pointer-coarse:`、`terminal:`、`reduce-motion:`。
- 平台变体：`ios:`、`android:`、`web:` 三类平台在 rank 中排在 Tailwind 原生变体之后。

### 3.4 网站自身的样式

`website/` 是 Astro 站点，使用 Tailwind v4（Vite 插件），并通过 `@fontsource-variable/figtree` 加载字体；Shiki 主题通过 `dark.theme.json` / `light.theme.json` 提供语法高亮配色。站点没有自定义 `.css` 或 SCSS 文件，样式集中在 Tailwind class 与少量 Astro 组件内联样式中。

## 4. 约定与约束

- **class 字符串只在 JS 侧拼接，解析与求值在 Rust 端完成**（`mod.rs` 注释：“JS only passes the string”）。不要在 JS 侧做 Tailwind class 合并——项目未引入 `tailwind-merge`，而是由 `ClassList::merged` 按 rank 排序后追加，保证“后面的 utility 优先”。
- **颜色统一使用 OKLCH**：默认主题所有色值均为 `oklch(...)`，palette 模块负责转 sRGB；CSS 解析器也支持 `oklch()`/`oklab()`（见 `style.rs` 的 `Color::parse`）。
- **未知 class 会被记录**：`ClassList.unknown()` 返回无法识别的类名，便于开发期告警。
- **交互状态的追踪是按需的**：只有当 class 中使用了相应变体（如 `hover:`、`focus:`）时，core 才向平台请求对应状态（`tracked_states()`），减少跨语言调用开销。
- **`currentColor` 继承链**：边框/描边等无色的值会回退到当前文本颜色，计算顺序为 `!` utility → inline → class → inherited → black（React Native 默认）。
- **主题变更的增量重算**：`Tailwind::set_tokens` 仅在 token 名称变化时清空缓存，仅值变化时只重算引用了 token 的 class（`deps::THEME`）。
- **网站构建工具链**：`website/package.json` 固定使用 `bun` 运行脚本（`bun scripts/play.js`），Astro 构建后再跑 `pagefind` 做站内搜索。

## 5. 适用性判断

该仓库是一个跨平台 UI 框架，前端样式并非传统 CSS 堆叠，而是 Rust 实现的 Tailwind 解析器 + JSON 设计令牌，因此 `frontend_style` 类别完全适用，且证据充分（核心模块、默认主题、网站 Tailwind 配置均存在）。