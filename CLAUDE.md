# CLAUDE.md

> **单一来源说明**:本仓库的开发指南统一维护在 [`WARP.md`](./WARP.md) 中。
> 为避免文档重复与不一致,本文件不复制任何内容,仅作为指向 `WARP.md` 的入口。

请直接阅读并遵循 [`WARP.md`](./WARP.md),其中包含:

- 构建与运行命令(`cargo run` 等)
- 测试命令(`cargo nextest` 等)
- Lint 与格式化(`./script/presubmit`、`./script/format`)
- 平台环境配置(`./script/bootstrap` 等)
- 架构总览(Rust workspace + WarpUI)

---

## 二开补充说明(本 fork 专用)

> 以下为本 fork 在 `WARP.md` 基础上的补充,记录与上游不同的本地约定。

- **开发与发布流程**:完整的分支模型、同步上游、发版步骤见 [`docs/FORK_WORKFLOW.md`](./docs/FORK_WORKFLOW.md)。
- **工作分支**:默认分支为 `dev`(二开在此进行);`master` 纯镜像上游;`release` 为发布分支。
- **平台**:本地开发环境为 Windows。`WARP.md` 中部分脚本(`run-clang-format.py`、`bootstrap`、C/Obj-C 相关)面向类 Unix / macOS,在 Windows 上使用时需注意适配。
