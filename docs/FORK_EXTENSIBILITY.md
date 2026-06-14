# Fork 扩展性调查 & 低侵入开发策略

> 目标:作为 fork,在**尽量不改上游 Rust 源码**的前提下扩展功能,从而减少未来从上游合并的冲突。
> 本文记录对 Warp(单体 Rust 应用 + 自研 WarpUI 框架,**无通用插件 SDK**)可扩展性的调查结果,以及按"侵入性"分层的开发策略。
> 调查日期:2026-06-14。文中文件路径相对仓库根。

## 一、核心结论

1. Warp 是**单体 Rust 二进制**,没有面向第三方的通用插件 SDK;UI/核心逻辑的扩展基本要改 Rust。
2. 但存在多个**数据驱动 / 外部进程**的扩展点,可零代码扩展。
3. 内置一个**窄用途的 JS 插件系统**(QuickJS),目前主要服务补全引擎。
4. **优先级:Tier 0(配置/MCP)> Tier 1(JS 插件)> Tier 2(新增 crate)> Tier 3(改上游文件)。** 能往上走一层,就少一份合并痛苦。

## 二、侵入性阶梯

### Tier 0 — 纯配置 / 数据驱动(零代码,零合并冲突)

从用户目录 `~/.warp/` 加载,**不在仓库里**,与上游合并永不冲突。

| 能力 | 配置位置 | 代码位置 | 默认可用 |
|------|---------|---------|---------|
| **MCP server**(给 AI agent 加工具/接外部服务) | `~/.warp/.mcp.json`(也支持项目级 `.mcp.json`、`.claude/.mcp.json` 等) | `app/src/ai/mcp/file_mcp_watcher.rs`、`app/src/ai/agent_sdk/mcp_config.rs` | ✅ `mcp_server` 在默认 features |
| Workflows(命令片段,支持 `{{参数}}`、shell 限制、热加载) | `~/.warp/workflows/*.yaml` | `app/src/workflows/local_workflows.rs` | ✅ |
| Themes 主题 | `~/.warp/themes/`(平台相关数据目录) | `app/src/themes/theme.rs` | ✅ |
| Settings / 快捷键(TOML,热加载) | `settings.toml`(平台相关) | `crates/settings/src/lib.rs`、`app/src/user_config/native.rs` | ✅ `SettingsFile` |
| Launch configurations | `~/.warp/launch_configurations/*.yaml` | `app/src/launch_configs/launch_config.rs` | ✅ |
| Tab configurations | `~/.warp/tab_configs/`(TOML) | `app/src/tab_configs/mod.rs` | 需 `TabConfigs` feature |

> **MCP 是被低估的王牌**:任何语言写的外部进程,挂到 `.mcp.json` 即可给内置 AI agent 加工具,对源码零改动。若二开多为"给 AI 加能力",这条能吃掉很大一部分需求。
> 配置支持环境变量模板 `${VAR_NAME}`。

### Tier 1 — JS 插件(`plugin.js`)— 几乎零侵入

- 插件为纯 JS,放 `~/.warp/plugins/<名>/plugin.js`,运行在 QuickJS(`rquickjs`)上,**不碰上游 Rust 源码**。
  - 加载逻辑:`app/src/plugin/host/native/mod.rs`(`PLUGIN_PATH_SUFFIX = ".warp/plugins"`,扫描各目录的 `plugin.js`)。
  - 架构:独立子进程 `plugin_host`,经 IPC(本地 socket)与主应用通信。模块在 `app/src/plugin/{mod,app,host,service}`。
- **唯一源码代价**:把 `plugin_host` 加入编译 features(定义见 `app/Cargo.toml:800`:`plugin_host = ["dep:rquickjs","dep:warp_js","warp_cli/plugin_host"]`)。**默认 features 不含它**,需追加 —— 这是一行追加式改动,合并冲突风险极低(详见下方"启用插件")。
- ⚠️ **能力边界(待深挖)**:目前该系统主要用于跑补全引擎(completions v2)。它能 hook 的扩展点可能较窄,不一定支持任意 UI/功能。完整插件 API 面尚未摸清——若要走 Tier 1,需进一步调查 `app/src/plugin/service` 和 `app/src/plugin/app`。

### Tier 2 — 新增独立 crate / 新文件(追加式 Rust,低冲突)

插件做不到、必须写 Rust 时,**优先"加文件"而非"改文件"**:
- 在 workspace 新建自有 crate(如 `crates/myfork_xxx`),逻辑都在里面。
- 上游文件里只留**极少数调用点**(理想 1–2 处),并用**自有 feature flag** 包起来。
- 合并冲突只可能在那几个调用点,可控易找。

### Tier 3 — 直接改上游现有文件(高冲突,尽量避免)

改 UI / 核心逻辑常逃不掉。降痛技巧:
- 每处改动用 **feature flag 隔离**(`crates/warp_features/src/lib.rs` 有 200+ flag 先例),fork 行为可开关、便于 review。
- 改动**集中、最小化**,加显眼标记注释(如 `// FORK:`)。
- 维护 `docs/FORK_PATCHES.md` 记录所有"改了上游文件"的清单,合并时优先盯这些。
- 开启 `git rerere`(自动复用冲突解决);**勤同步上游**(小步合并比攒一大坨痛苦小)。

## 三、必须改源码的功能(已知)

- **内置命令补全签名**:`crates/command-signatures-v2/`(TS 源 + `rust-embed` 嵌入二进制),新增需重编;`completions_v2` feature 默认未开。
  - 零侵入替代:依赖系统 shell 的原生补全(bash `_*` 函数、zsh compdef 等)。
- **启动钩子**:无内置 `.warprc`/启动脚本机制;只能用 shell 自身的 `~/.bashrc` / `~/.zshrc`。
- **终端 UI / 渲染 / 交互改动**:需改 WarpUI 源码(Tier 3)。

## 四、Feature Flags(对 fork 隔离改动很有用)

- 定义:`crates/warp_features/src/lib.rs`;声明:`app/Cargo.toml`(约 165 个 features,`default = [...]` 在 ~L500)。
- 默认集**含** `mcp_server`;**不含** `plugin_host`、`completions_v2`。
- 编译期 feature 需改 Cargo.toml;部分为运行时 flag(经 `UserPreferences` / `FeatureFlag::set_enabled()`,无需重编)。
- **fork 用法**:把自有改动包在自定义 feature 后,既隔离又可开关。

## 五、启用插件 / 自定义 feature 的低侵入做法

发布走 `oss` channel,`script/windows/bundle.ps1` 里 oss 的 `cargo build` **未传 `--no-default-features`**,因此默认 features + oss 追加的 `release_bundle,gui,nld_*` 都生效。要为 fork 开启额外能力(如插件),两种低侵入方式:

1. 在 `bundle.ps1` 的 oss 分支 `$FEATURES` 里追加 `plugin_host`(改打包脚本,不碰核心 Rust)。
2. 或在 `app/Cargo.toml` 定义一个 fork 专属 feature(如 `myfork = ["plugin_host", ...]`)再于打包时启用 —— 改动集中在一处。

> 两种都属"追加式",合并冲突面极小。

## 六、用户数据目录速查

```
~/.warp/
├── .mcp.json                 # MCP 配置(也支持项目根 .mcp.json)
├── workflows/*.yaml          # 工作流
├── themes/                   # 主题
├── launch_configurations/    # 启动配置
├── tab_configs/              # Tab 配置(需 feature)
├── plugins/<名>/plugin.js    # JS 插件(需 plugin_host feature)
└── skills/                   # Agent 技能脚本
settings.toml                 # 设置(平台相关:~/.warp 或 ~/.config/warp 或 %APPDATA%\warp\WarpOss\config)
```

> OSS channel 的数据目录是独立的(如 macOS `~/.warp-oss/`、Windows `%APPDATA%\warp\WarpOss\`),与官方 Warp 不冲突。

## 七、待办 / 后续调查

- [ ] 深挖 Tier 1 插件 API 面(`app/src/plugin/service`、`app/src/plugin/app`),确认除补全外还能 hook 什么。
- [ ] 确定具体二开需求后,逐功能定位落在哪一层、是否需改源码。
