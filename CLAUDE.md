# RustDesk (ojbkxc fork) 开发者指南

## 项目概览

本仓库是 **RustDesk 官方仓库的二次开发 fork**（远程桌面客户端，Rust + Flutter），
用于自定义客户端开发，配套自建 hbbs/hbbr 服务器与 lejianwen API 服务器。

- **远端**：https://github.com/ojbkxc/rustdesk（master 直接 push，无 PR 流程）
- **上游**：rustdesk/rustdesk（upstream remote 已配置；同步官方必须先问）
- **基线**：官方 master（1.5.0 开发线，2026-09-18 与官方同步）

## 核心铁律：本地不编译，验证只走 GitHub CI

本机（Windows）没有 vcpkg + Flutter + Rust 完整工具链，本地编译结果不可信：

1. **禁止** `cargo build/check/clippy/test`、`build.py`、`flutter build` 等一切本地构建命令
2. 改动完成 → push master → 盯 Actions：
   - 小改 Rust → `CI` workflow（单 linux job，最快）
   - 大改 / UI / 发布 → `Full Flutter CI`（全平台矩阵）
3. CI 查询（本机无 gh）：`curl -s "https://api.github.com/repos/ojbkxc/rustdesk/actions/runs?head_sha=<SHA>"`
4. **完成态定义 = CI 绿**。红 CI 不许结束回合，修根因重推（同一修复盲试 ≤2 次后停下报告）

## 目录速览

| 路径 | 内容 |
|---|---|
| `src/` | Rust 核心：连接 (`rendezvous_mediator.rs`)、会话、服务 (`src/server/`) |
| `flutter/` | 当前 UI（Dart）；`src/ui/` 为废弃 Sciter UI |
| `libs/hbb_common/` | **git submodule**，与服务端共享（proto/Config），改动慎之 |
| `libs/base/` | 客户端专属库（option keys / 消息 proto / 文件传输），新逻辑优先放这里 |
| `libs/scrap/` `libs/enigo/` `libs/clipboard/` | 截屏 / 输入模拟 / 剪贴板 |
| `src/lang/` | 翻译表；`template.rs` 是主键列表，永不编辑 |
| `.github/workflows/` | CI 定义（见下） |

## CI 说明

- 触发：push master（`ci.yml` 忽略 docs/README/res 等纯文档路径）或手动 dispatch
- **fork 无官方签名 secrets**（MACOS_P12_*/ANDROID_SIGNING_KEY 等）：签名相关 job 失败
  或产出未签名包属预期，不是代码问题
- `Full Flutter CI` 曾因 runner label（`windows-11-arm`、`macos-15-intel`）+ secrets 缺失
  startup_failure；裁剪矩阵需先问
- Android/macOS/iOS 产物未签名，直接安装会失败，属预期

## 开发规范

完整规则见 [AGENTS.md](./AGENTS.md)。要点：

1. **官方 AGENTS.md 规则全部继承**——最小 diff、纯增量、一个 crate 一条 use、
   注释只写为什么、回归面收尾检查
2. submodule（hbb_common）不轻易动，客户端专属逻辑进 `libs/base`
3. 只 add 自己改的文件，禁止 `git add -A`
4. 同步官方（fetch upstream / cherry-pick）必须先问
5. push 直接 `git push origin master`（origin 已内嵌 token），严禁回显含 token 的 URL

## 常见问题

### Q: 为什么本地编译会失败？
A: 缺 vcpkg/Flutter 工具链，且 GitHub CI 环境版本全部 pin 死（Rust 1.75/Flutter 3.24.5/
vcpkg 指定 commit），本地复刻这套环境没有意义。官方 CI 是唯一权威。

### Q: 如何同步官方最新代码？
A: 先问用户。然后 `git fetch upstream`，挑选官方 tag cherry-pick，不要 merge upstream/master。

### Q: 改了 UI 没生效？
A: UI 在 `flutter/lib/`（Dart），不是 `src/ui/`（那是废弃的 Sciter）。改协议层才是 Rust。

### Q: CI 里 Android/macOS 签名失败怎么办？
A: fork 没有官方的签名 secrets，属预期。要签名产物需自行配置 secrets 或接受未签名包。
