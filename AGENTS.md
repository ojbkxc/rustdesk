# AGENTS.md — RustDesk (ojbkxc fork)

> 给 AI 编码代理的仓库指南。规则分三级：**Never(禁止) / Ask first(先问) / 默认自主**。
> 每条硬规则都对应真实踩过的坑，不要试图绕过。
> 本文件在官方 AGENTS.md（上游 RustDesk 代理规则）基础上叠加本仓库的私有开发纪律；
> 官方的 Rust/Tokio/最小 diff/翻译规则继续有效，冲突时以本文件为准。

## 0. 项目速览(新会话必读)

**RustDesk** 是开源远程桌面客户端（Rust 核心 + Flutter UI），本仓库是
**ojbkxc/rustdesk** —— 基于官方 `rustdesk/rustdesk` master（1.5.0 开发线）的二次开发 fork。

- **仓库**：https://github.com/ojbkxc/rustdesk（fork，master 分支，直接 push，不走 PR）
- **上游**：https://github.com/rustdesk/rustdesk（upstream remote 已配置，用于同步官方）
- **基线**：master 跟随官方 1.5.0 开发线（Cargo.toml `version = "1.5.0"`）
- **服务端**：hbbs/hbbr 用官方 `rustdesk/rustdesk-server` OSS 版或 lejianwen fork（forapi 分支），
  配套 API 服务器 `lejianwen/rustdesk-api`（源码在 `D:\GitHub\rust\rustdesk-api-master\`）
- **代码结构**：
  - `src/` Rust 核心（连接/会话/服务端内 audio/video/input/clipboard）
  - `flutter/` 当前 UI（Dart），`src/ui/` 是已废弃的 Sciter UI
  - `libs/hbb_common/` **git submodule**，与服务端共享（rendezvous proto、Config）
  - `libs/base/` 客户端专属代码（option keys、消息 proto、文件传输）
  - `libs/scrap/` 截屏；`libs/enigo/` 输入模拟；`libs/clipboard/` 剪贴板
  - `src/lang/*.rs` 各语言翻译（`template.rs` 是主键列表，**永不编辑**）

## 1. Never(硬性禁止，违反即返工)

1. **禁止本地编译**。不运行 `cargo build` / `cargo check` / `cargo clippy` / `cargo test` /
   `build.py` / 任何 Flutter 构建命令。本机没有 vcpkg + Flutter + Rust toolchain 的完整验证
   环境，本地结果不可信。**唯一编译验证是 GitHub Actions**（push 到 master 或手动 dispatch）。
2. **禁止本地直接改 submodule**（`libs/hbb_common`）并让它脱离官方指向——改 submodule 指针
   会导致 CI checkout 与本地不一致。客户端专属代码放 `libs/base`。
3. **禁止为绕过 CI 而修改 workflow**（`.github/workflows/`）——除非任务就是修 CI 且用户明确要求。
4. **禁止 `git add -A` / `git add .`**：只 add 自己本次改动的文件（可能存在并行会话的未提交改动）。
5. **禁止单独编辑 `src/lang/template.rs`**（翻译工作只填各语言文件的空值）。
6. **禁止把服务器地址、密钥、token 写进任何入库文件**；push 用 .git/config 内嵌 token
   （已配置），严禁回显含 token 的 URL。
7. **禁止合并 upstream/master 进本仓库 master**——官方 master 每天几十个 commit，
   混入会引入大量未经审查的变更。同步官方走 explicit 的 `git fetch upstream` + 挑选
   `git cherry-pick <tag 范围>`，且同步前必须问用户。

## 2. CI 结构（编译验证的唯一途径）

### 触发方式

| Workflow | 触发 | 用途 |
|---|---|---|
| `ci.yml` | push master（忽略 docs/README/res 等路径）+ workflow_dispatch | Rust 库编译验证（x86_64-linux matrix job），**最快信号** |
| `flutter-ci.yml` | push master + workflow_dispatch | 全平台 Flutter 构建矩阵（Windows/macOS/Linux/Android/iOS/Web），最重 |
| `flutter-build.yml` | 被上面两个 `uses:` 的可复用 workflow | 实际构建逻辑，16 个 job |
| `flutter-nightly.yml` / `flutter-tag.yml` | 定时 / tag | 产物发布，日常开发不碰 |

- **fork 仓库的 Actions 默认禁用**——push 空提交激活（已激活，total 11 个 workflow）。
- 日常改 Rust 代码看 `CI`（单 job，~20 分钟内出结果）；改 Flutter/Dart 或发布前跑 `Full Flutter CI`。
- `Full Flutter CI` 曾报 `startup_failure`：其矩阵含 `windows-11-arm`、`macos-15-intel` 等
  runner label + macOS 签名 secrets（`MACOS_P12_*`）在 fork 中缺失。**允许给无关平台 job 加
  `if: false` 或裁剪矩阵来让 CI 变绿**（仅限 job 级裁剪，不许动构建步骤本身），改 workflow 前先问用户。
- Android/macOS 签名 job 在无 secrets 时会失败或产出未签名包，属预期，不算编译问题。

### CI 查询（本机无 gh CLI）

```
curl -s "https://api.github.com/repos/ojbkxc/rustdesk/actions/runs?head_sha=<SHA>&per_page=5"
```

Token 从 `D:\GitHub\llm-proxy\.git\config` 的 remote URL 提取（或本仓库 .git/config）。

## 3. 开发方法

- **官方 AGENTS.md 规则全部继承**：最小 diff、纯增量优先（`#[cfg]`-gated 新块不动老代码）、
  一个 crate 一条 `use`、注释只写为什么、手术式修改、回归面检查（改动收尾时列出所有
  被改的既有文件并说明为何不可避免）。
- **hbb_common 是 submodule**：改它会波及服务端，能放 `libs/base` 就放 `libs/base`。
- **UI 改动 = Dart**（`flutter/lib/`），协议/连接层 = Rust（`src/`）。先想清楚改动归属层再动手。
- **commit 信息**：conventional 前缀 + 中文/英文与历史一致（官方历史是英文，本仓库私有提交用中文）。
- 提交前只做静态自检（读 diff 对照官方 AGENTS.md 规则），编译问题留给 CI。

## 4. 自动迭代闭环

```
理解任务 → 就绪判定 → 手术式实现 → push master → 盯 CI（绿=完成）
              ↑                                  │
              └──── 修根因，同一修复盲试 ≤2 次 ←── CI 红
```

1. **就绪判定**：非平凡改动先写简短方案（目标/取舍/放弃项）再动手。
2. **push 方式**：本仓库 origin 已内嵌 token，直接 `git push origin master`；
   禁止交互式凭据管理器（无人值守会永久挂起）。
3. **CI 失败处理**：只修根因；同一修复盲试不超过 2 次，仍红则停下报告并附 run URL。
4. **不许宣称被阻塞**：限制是"从真实失败挣来的结论"；没实际尝试就说 "not attempted"。
5. **结束卫生**：不留未提交改动；最终答复带 CI 结论或 commit SHA。

## 5. Ask first(先问再做)

- 同步官方更新（fetch upstream / cherry-pick 官方 tag / submodule bump）。
- 动 `.github/workflows/`（含裁剪 job 矩阵让 fork CI 变绿）。
- 升级 Rust/Flutter/vcpkg 版本（workflow env 里全部 pin 死，牵一发动全身）。
- 新增 Rust crate / Dart package 依赖。
- 改 `src/rendezvous_mediator.rs`（协议层，影响与服务端互通）。
- 任何删除超过 100 行的批量清理。

## 6. 参考文件

- 官方代理规则（Rust/Tokio/最小 diff/翻译细则）：`AGENTS.md` 官方部分 / `GEMINI.md`
- 构建细节：官方 `README.md` 与 `.github/workflows/flutter-build.yml`（各平台依赖、版本 pin）
- 生态源码：`D:\GitHub\rust\rustdesk-api-master\`（lejianwen API 服务器）、
  `D:\GitHub\rust\rustdesk-server-forapi\`（lejianwen fork 服务端）
