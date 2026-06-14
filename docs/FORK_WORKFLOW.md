# Fork 开发与发布工作流

> 本文档是 **TheMcSwift/warp** 这个 fork 的开发/发布运维手册。
> 它记录的是本 fork 与上游 warpdotdev/warp 不同的**流程约定**,与代码无关。
> 通用开发命令(构建/测试/lint)见 [`../WARP.md`](../WARP.md)。

## 远端约定

- `origin` = `TheMcSwift/warp`(你的 fork)
- `upstream` = `warpdotdev/warp`(官方)
- ⚠️ `gh` 命令一律加 `-R TheMcSwift/warp`,否则会默认指向 upstream。

## 分支模型

```
upstream/master ──sync──► origin/master(纯镜像,不放任何自定义文件 → 同步零冲突)
                              │ 定期 merge 进
                              ▼
                            dev(默认分支:日常开发 + 自定义 workflow)
                              │ 完成一批需求后 PR(release 已设保护,强制走 PR)
                              ▼
                          release ──push 触发──► 自动打包 Windows(oss) → fork Releases 发 prerelease
```

- **`master`**:只镜像上游,**永不在其上提交**。
- **`dev`**:默认分支,所有开发和 fork 专属文件(CLAUDE.md、release-fork.yml、本文档)都在这里。
- **`release`**:从 dev 拉,合入即触发发布。

---

## A. 同步上游到 master(日常)

master 是纯镜像,只做快进合并:

```bash
git checkout master
git fetch upstream
git merge --ff-only upstream/master    # 纯镜像,必定能快进
git push origin master
```

> 若 `--ff-only` 报错,说明 master 被意外改过——**别强推**,先排查。正常永不发生。

## B. 上游 workflow 的处理(通常无需操作)

fork 继承的上游 workflow(repo-sync、populate_build_cache、ci 等)在 fork 上默认**休眠**,不会运行,也不会在同步时报红。`gh workflow list -R TheMcSwift/warp` 只会显示 `Release (fork)` 和 `Dependency Graph` 为 active。

- ⚠️ **不要**在 GitHub 仓库 Actions 页点 "Enable all workflows" —— 那会激活所有休眠的上游 workflow,它们才会开始跑红。
- 仅当你看到 `repo-sync` / `populate_build_cache` 在 Actions 里跑红时,才执行(此时它们已注册,命令才会成功):

```bash
gh workflow disable repo-sync.yml -R TheMcSwift/warp
gh workflow disable populate_build_cache.yml -R TheMcSwift/warp
```

## C. 把上游更新并入 dev(每次跑完 A 之后)

```bash
git checkout dev
git merge master                       # 把上游新代码并进 dev
# 如有冲突:编辑解决 → git add <文件> → git commit
git push origin dev
```

> 冲突只会出现在你改过、上游也改过的同一文件。自定义文件上游没有,通常不冲突。

## D. 日常开发

```bash
# 方式1:直接在 dev 上开发
git checkout dev
# ...改代码...
git add -A && git commit -m "feat: xxx"
git push origin dev

# 方式2:feature 分支(大功能推荐)
git checkout dev && git checkout -b feature/xxx
# ...改代码、提交...
git push -u origin feature/xxx
gh pr create --base dev --head feature/xxx -R TheMcSwift/warp
```

本地自检(代替 CI):

```bash
./script/presubmit        # fmt + clippy + 测试(在 Git Bash 里跑)
```

## E. 发版(dev → PR → release → 自动发布)

```bash
# 1. 开 PR:dev → release(release 已设保护,必须走 PR)
gh pr create --base release --head dev -R TheMcSwift/warp \
  --title "Release: <这批需求简述>" --body "本次发布内容..."

# 2. 合并 PR(触发 release-fork.yml 自动构建 + 发布)
gh pr merge <PR编号> --merge -R TheMcSwift/warp
```

合并后自动:打包 Windows(`oss` channel,无 Sentry)→ 在 fork Releases 发 prerelease,tag 形如 `fork-v20260614.5`。

自定义版本号 / 手动重发:

```bash
gh workflow run release-fork.yml -R TheMcSwift/warp --ref release -f tag=v1.0.0
```

**快速验证模式**(只验证流水线、不要正式产物时):勾选 `fast=true` → 用 debug profile(无 LTO),比 `oss` LTO 略快;但仍是冷编译,见下方"构建耗时"。

```bash
gh workflow run release-fork.yml -R TheMcSwift/warp --ref dev -f fast=true
```

### 构建耗时与缓存

- Warp 是超大 workspace,**首次冷编译约 50 分钟**(免费 runner + 几百个 crate)。
- 流水线已加 `Swatinem/rust-cache`(固定 `shared-key`、任何分支都保存),所以**之后重复构建只编增量,会快很多**。第一次构建仍是冷编译(同时负责把缓存填上)。
- 上游 `prepare_environment` 自带的 rust-cache 只在 master 保存,对 fork 分支无效——这正是额外加缓存的原因。
- 不要用 `windows-latest-large` 等大型 runner 提速:**大型 runner 即使公开仓库也计费**。
- ✅ 流水线已端到端验证通过,产物为 `WarpOssSetup.exe`(oss channel,独立应用名 WarpOss,不含 Sentry)。

## F. 监控与排错

```bash
gh run list --workflow="Release (fork)" -R TheMcSwift/warp   # 发布构建列表
gh run watch <run-id> -R TheMcSwift/warp                     # 实时盯一次运行
gh run view <run-id> --log-failed -R TheMcSwift/warp         # 失败时看日志
```

---

## 发布流水线说明

- 文件:[`.github/workflows/release-fork.yml`](../.github/workflows/release-fork.yml),fork 专属,只存在于 dev/release(不在 master,故不影响上游同步)。
- 复用底层打包脚本 `script/windows/bundle.ps1` 和上游 composite action `prepare_environment`(对 fork 安全:跳过私有 SSH/channel-config,protoc 用 winget 兜底)。
- 用 `oss` channel:不含 Sentry/崩溃上报。
- **暂不复用上游 `create_release.yml`**:它依赖 Warp 私有基础设施(GCS、Sentry、Apple 公证、自建 runner、私有 channel-config 仓库),fork 无法运行。
- 平台目前只发 **Windows**;macOS 需 Apple 开发者证书+公证,Linux 可后续扩展。
