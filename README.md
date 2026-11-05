# dsh-nas · DeepSeek Harness NAS 打包

> **本仓是自建二开副本**：`NeeYoonc/dsh-nas`，基于 [EIGHTfs/DeepSeekHarness-NAS](https://github.com/EIGHTfs/DeepSeekHarness-NAS) 的群晖 Synology DSM `.spk` / 飞牛 fnOS `.fpk` 原生打包工程。
> 上游智能体框架 DeepSeek Harness（dsh）由 [DeepSeek AI](https://github.com/deepseek-ai/deepseek-harness) 开源。
> 来源与派生关系见 [NOTICE](NOTICE)，许可见 [LICENSE](LICENSE)。

- **本仓 Release**：https://github.com/NeeYoonc/dsh-nas/releases
- **上游仓库**：https://github.com/EIGHTfs/DeepSeekHarness-NAS
- **套件名 / 安装路径保持 `DeepSeekHarness-NAS`**（与上游一致）：安装目录、服务名、数据目录都不变，可沿用现有 NAS 数据；仅代码与发布渠道由本仓独立维护。

DeepSeek Harness（dsh）的 NAS 适配包：群晖 DSM `.spk` / 飞牛 fnOS `.fpk`，在 NAS 原生运行，无需容器。

## 功能

| 功能 | 做什么 | 入口 |
|---|---|---|
| 套件安装 | 群晖装 spk、飞牛装 fpk，装完即用 | 套件中心 |
| Web UI | 多模型配置、自定义 OpenAI 兼容端点 | `http://<NAS-IP>:30800`（spk）/ `:3080`（fpk） |
| 反向代理 | 反代端口对外，dsh 原生端口仅本机 | `http://<NAS-IP>:<反代端口>` |
| 门户免密 | 套件图标打开带 token，认证后免密 | DSM/fnOS 桌面图标 |
| 局域网限制 | 公网 IP 一律 403 提示页 | 内置，无需配置 |
| 命令行 `dsh` / `pnpm` | 装完 SSH 直接敲，无需进套件目录 | `/usr/bin/dsh`、`/usr/bin/pnpm` |
| 网页安装工具 | 已独立为单独仓库（远程装/卸/检/修，带历史） | `nas-pkg-installer` |
| 自动构建发布 | 官方更新自动对齐构建，发 Release 四产物（x86_64/arm64 × spk/fpk） | GitHub Actions |
| 架构参数化 | 同一脚本出 x86_64 / arm64 包 | `build/build-config.yaml` 的 `defaults.arch` |

## 访问地址与端口

| 平台 | 反代端口（唯一入口） | dsh 原生端口（仅本机） | 容器页端口 |
|---|---|---|---|
| 群晖 DSM（spk） | 30800 | 30801 | 30802 |
| 飞牛 fnOS（fpk） | 3080 | 3081 | 3082 |

权威值在 `build/build-config.yaml` 的 `defaults` / `spk` / `fpk` 三段；改端口只改这里。

```bash
http://<NAS-IP>:30800     # 群晖入口（<NAS-IP> 用 `hostname -I` 取主网卡局域网地址）
http://<NAS-IP>:3080      # 飞牛入口；容器页：群晖 30802 / 飞牛 3082
./start.sh --proxy-port 31800 --dsh-port 31801 --container-port 31802 start   # 运行时改端口（写入 <数据目录>/<版本>/ports，跨升级保留）
```

## 安装与卸载

- 群晖：套件中心 → 手动安装 → 选 `.spk`；DSM 7.0-40851 起，x86_64。
- 飞牛：应用中心 → 手动安装 → 选 `.fpk`；fnOS 1.1.0 起。
- 安装向导可留空端口（用打包默认）；卸载向导默认**保留数据**，选「彻底删除」才清数据。
- 卸载走 Web UI，命令行卸载会被平台拒绝。
- 回退旧版：Web UI 卸载（保持默认「保留数据」）→ 装旧版包 → 直接启动。数据按版本目录隔离，各版本用各自的配置与会话。

## 账号登录（仅回环地址可用）

**症状**：用局域网 IP 或域名打开 Web UI，点「登录」一直弹「登录失败 / 操作未完成，请重试。」（只有「添加 API Key」「重新登录」两个按钮）；同一账号在 Windows 桌面端登录正常。

**原因**：DSH 账号登录（Platform OAuth）只接受**回环 HTTP origin**（`localhost` / `127.0.0.1` / `[::1]`）作为回调地址。经反代、域名、HTTPS 或 `--trusted-host` 访问时，登录入口仍显示，但发起登录必然进入通用失败重试状态 —— 这是 DSH 的设计限制，**不是本包的问题**，换域名/上 HTTPS 也绕不过。

**处置一：一次性回环登录（想保留账号登录时用）**

Windows 管理员 PowerShell（不需要 NAS 开 SSH）：

```powershell
netsh interface portproxy add v4tov4 listenaddress=127.0.0.1 listenport=30800 connectaddress=<NAS-IP> connectport=30800
```

浏览器打开 `http://127.0.0.1:30800`（飞牛把 `30800` 换成 `3080`）完成登录；登录态保存在 NAS 上的那个 DSH 实例里，之后删掉转发、日常继续用 `http://<NAS-IP>:30800`：

```powershell
netsh interface portproxy delete v4tov4 listenaddress=127.0.0.1 listenport=30800
```

Linux/macOS 等价做法是 SSH 隧道（需 NAS 的 sshd 允许转发）：

```bash
ssh -N -L 30800:127.0.0.1:30800 <NAS用户>@<NAS-IP>
```

若报 `channel 2: open failed: administratively prohibited`，说明 NAS 侧 sshd 禁止了 TCP 转发（DSM 默认如此），改用上面的 `netsh`。

**处置二：改用 API Key（长期 / 多设备推荐）**

点弹窗里的「添加 API Key」，或按下方「配置模型」直接写配置文件。代价是拿不到账号登录才有的余额 / 赠送额度，但模型可正常使用。

> 完整判据与更多细节见 [`docs/故障排查指南.md`](docs/故障排查指南.md) §2.5。

## 常用操作（start.sh）

```bash
./start.sh start        # 启动（拉起 dsh → 反代 → 容器页 → 校验端口）
./start.sh stop         # 停止；卸载流程下输出「已卸载」
./start.sh restart      # 重启（未启动时相当于 start）
./start.sh status       # 查状态：0=运行中，3=未运行
./start.sh fix-deps     # 启动报缺包时的补齐兜底
./start.sh gen-portal --token   # 重新生成门户入口并写入当次 token
```

```bash
synopkg start DeepSeekHarness-NAS               # 群晖启停（stop/restart/status 同名）
appcenter-cli start DeepSeekHarness-NAS         # 飞牛启停（stop/status/check 同名）
```

## 配置模型

Web UI 设置 → Models → 添加提供方；或直接改文件。

```
/var/packages/deepseek-harness-nas/target/var/data/.dsh/settings.yaml           # 群晖 provider 配置
/var/packages/deepseek-harness-nas/target/var/data/.dsh/.credentials.yaml       # 群晖 API Key
```

```yaml
providers:
  myprovider:
    id: myprovider
    name: My Provider
    type: openai-completions
    base_url: https://api.example.com/v1      # Bearer Token 类端点
    models:
      - id: my-model
        name: my-model
API_KEYS:
  DEEPSEEK_API_KEY: "<你的 API 密钥>"
```

## 构建与发布

```bash
./build/build-common.sh                  # 公共预编译 → target（全量约 15-20 分钟）
./build/build-common.sh "" 1             # 复用已有 target，秒级
./build/SPK/pack-spk.sh                  # → build/staging/DeepSeekHarness-NAS_x86_64-<版本>.spk
./build/FPK/pack-fpk.sh                  # → build/staging/DeepSeekHarness-NAS_x86-<版本>.fpk
./scripts/promote-release.sh             # 验证通过的产物提升到 release/
```

发版走 GitHub Actions：推 tag、手动 `workflow_dispatch`（有 `arch`、`dsh_tag` 入参）、或由看门狗自动对齐官方 tag。

**通道口径（本仓自定）**：

- **只发正式版** —— 拉源码时自动跳过上游 `alpha` / `beta` tag（`DSH_ALLOW_PRERELEASE=1` 可临时放开），正式版 = `rc` 及以上（上游无 stable tag，`rc` 即最成熟通道）。
- **架构成对** —— 正式版按 `x86_64` 与 `arm64` 各派发一次，同一 Release 下产出 **4 个产物**：`*_x86_64-<ver>.spk`、`*_x86-<ver>.fpk`、`*_armv8-<ver>.spk`、`*_arm-<ver>.fpk`。
- **跟随上游** —— 看门狗每 30 分钟比对上游最新正式 tag 与本仓最新 Release tag；「不相等 + 无构建在跑」即自动派发双架构重建（**成功不节流**，上游一发新版就追，无冷却）；仅当同一版本连续失败 ≥ 3 次时暂停，等上游下一版或人工修复。


## 脚本与环境变量索引

### 构建与打包

| 名称 | 作用 |
|---|---|
| `build/build-common.sh` | 公共预编译（install + build + 裁剪 → target）；参数 `[SRC] [SKIP_BUILD] [--dry-run]` |
| `build/build-lib.sh` | 打包公共函数库（`gen_start_sh()`、`resolve_arch()`） |
| `build/prune-target.sh` | 白名单裁剪，三模式：`--before-install <SRC>` / `--node-modules <SRC>` / `<TARGET> [WHITELIST]` |
| `build/gen-prune-whitelist.sh` | 从 npm 锁文件生成 `lockfileDeps`；`[锁文件]` / `--dry-run` |
| `build/build-npm-app.sh` | FPK npm 链路应用体构建；`[VERSION]`，可空 = 取当前版本 |
| `build/build-test-fpk.sh` | 测试版 FPK 构建；无参数 |
| `build/fix-runtime-deps.sh` | 打包期运行时依赖补齐；`<TARGET> <BUILD_SRC> <NODE> [--max-rounds N]` |
| `build/fix-node-links.py` | pnpm 三层链接补全与错链纠正；`<应用体或 target> [--quiet]` |
| `build/prune_common.py` | 裁剪公共模块（依赖闭包计算）；由 `PRUNE_COMMON_DIR` 定位 |
| `scripts/learn-prune-whitelist.sh` | 白名单自动学习（`--scan <SRC>` / `--log <FILE>`），写入 `_autoLearned` |
| `scripts/prepare-build-env.sh` | 构建环境准备（幂等、不 mount）；无参数 |
| `scripts/fetch-dsh-latest.sh` | 拉官方源码到 `src/deepseek-ai/<tag>`；`[--token <ghp>]`，`--print-tag` 只出 tag |
| `scripts/fetch-release-mt.sh` | 下载本仓 Release 资产；`[--tag <tag>] [--only spk\|fpk] [--threads N]` |
| `scripts/promote-release.sh` | staging → release 提升；`D_REL=<dir>` 可覆盖输出目录 |
| `scripts/prune-release-assets.py` | 清理 Release 改名残留资产；`--tag <tag> --keep-from <dir> [--dry-run]` |
| `scripts/update-release-note.sh` | 单独刷新某个 Release 正文（不重新构建）；`--tag <tag> [--trigger <名>] [--dry-run]`，`--tag` 可空=最新 |
| `scripts/collect-failure-fixtures.py` | 采集历史 CI 失败日志做回归夹具；`[--limit N] [--out <dir>] [--dry-run]` |
| `scripts/fetch-official-docs.py` | 抓官方文档快照；无参数 |
| `scripts/generate-diff-report.sh` | 正式版 / 测试版 FPK 差异报告；无参数 |

### 运行与维护

| 名称 | 作用 |
|---|---|
| `scripts/dsh` / `scripts/pnpm` | dsh、pnpm 命令包装器（打进包 `bin/`） |
| `scripts/fix-login-shell.sh` | 登录 shell 悬空探测修复；`--check` / `--patch [文件]` / `--passwd` / `--shell PATH` / `--dry-run` |
| `scripts/fix-pnpm-store.sh` | 修复 pnpm storeDir 记录；`<目标树>`，可空 = 当前实例 |
| `scripts/fix-dsh-settings-namespace.sh` | 修复 alpha 版 `settingsNamespace` 缺失导致的插件加载失败；无参数，幂等带备份 |
| `scripts/fix-dsh-bundle-patch-insert.py` | 修复 bundle patch 插入问题；`<目标树>` |
| `scripts/set-dsh-cpu-quota.sh` | 设置 DSH CPU 配额；无参数 |
| `scripts/verify-dsh-cpu-quota.sh` | 验证 CPU 配额生效；无参数 |
| `scripts/migrate-session.sh` | 会话跨版本迁移；`--list` / `--check <文件>` / `--fix-layout` / `--cwd <新cwd> --in <源> --id <id>` / `--rollback [备份名]` |
| `scripts/clean-dsm-residue.sh` | DSM 卸载残留清理；`<套件名> [主机] [SSH用户]`，`--keep-data`（缺省）/ `--delete-data` |
| `scripts/clean-build-artifacts.sh` | 清理失败/中间构建；`--caches` / `--dry-run` / `--force` |
| `scripts/lib/common.sh` | 全仓唯一公共函数库（日志、安全删除、端口读取、`running_dsh` 等） |

### 网页安装工具（已独立）

远程安装/卸载/检查 SPK/FPK 的网页工具已拆分为独立仓库，本仓不再包含其代码与配置：

- 仓库：https://github.com/EIGHTfs/nas-pkg-installer
- 用法见该仓库 README

本仓只负责打包与发布。

### 守卫与测试

| 名称 | 检查内容 |
|---|---|
| `scripts/check-readme-coverage.py` | 代码里的脚本名/开关名必须在 README 出现；`--list` 只列不失败 |
| `scripts/check-workflow-yaml.py` | `.github/**` YAML 结构、步骤缩进、`needs` 引用悬空 |
| `scripts/check-build-config.py` | `build/build-config.yaml` 的 `defaults`/`spk`/`fpk` 必须是映射；`--file <path>` / `--list` |
| `scripts/check-build-naming.py` | 命名规范 + 全部 `.sh`/`.py` 语法与可执行位 |
| `scripts/check-common-functions.py` | 公共函数不得在其它脚本重复定义 |
| `scripts/check-destructive-ops.py` | 敏感路径 `rm -rf` 必须 `--one-file-system` |
| `scripts/check-packaging-invariants.py` | 打包不变量（含生成函数自包含、按声明范围选版本、出货裁剪并入 `_autoLearned`）；`--verbose` 列出每条检查 |
| `scripts/check-package-parity.py` | 两包结构比对（软链/悬空/manifest）；`<基准包> <候选包> [--verbose] [--baseline-self]` |
| `scripts/check-fpk-payload.py` | FPK 载荷（`app.tgz` 不含软链、权限）；`<包路径> [--expect-pkg <名>]` |
| `test/learner-dryrun.test.sh` | 白名单学习器 dry-run 回归；`bash test/learner-dryrun.test.sh` |
| `test/prune-release-assets.test.sh` | Release 资产清理回归；`bash test/prune-release-assets.test.sh` |
| `test/safe-rm-rf.test.sh` | 含挂载点目录绝不被删的回归；`bash test/safe-rm-rf.test.sh` |

### 环境变量

| 变量 | 默认 | 含义 |
|---|---|---|
| `PRUNE_BEFORE_INSTALL` / `PRUNE_BEFORE_BUILD` | `1` | install 前剥非白名单 devDeps / build 前裁 `.pnpm`；`0` 关闭 |
| `PRUNE_COMMON_DIR` | 脚本同级 | `prune_common.py` 所在目录 |
| `BUILD_STAGE` / `SKIP_BUILD` | `all` / `0` | 分阶段（`install`/`build`/`prune`/`all`）；`SKIP_BUILD=1` 复用已有 target |
| `NPM_MODE` / `DRY_RUN` | `0` / `0` | FPK 走 npm 链路（产物后缀 `-npm`）/ 只打印计划不执行 |
| `DSH_PROXY_PORT` / `DSH_VERSION` | `30800` / 空 | 反代端口覆盖（等价 `--proxy-port`）/ 指定拉取的官方源码版本 |
| `DSH_NODE_DIST` / `DSH_GIT_BIN` / `DSH_TSC_MEM` | 自探测 / 自探测 / 可用内存 75% | node 发行版目录 / git 可执行路径 / tsc 堆上限（MB） |
| `DSH_NPM_REGISTRY` / `DSH_TOKEN_FILE` / `DSH_GIT_PUSH_CONFIG` | 国内镜像 / 自动探测 / 工作区 `config.json` | 补包 registry / GitHub token 文件 / 取 githubToken 的配置路径 |
| `DSH_KEEP_DATA` / `DSH_UNINSTALLING` / `DSH_DEPS_RETRIED` | `1` / 空 / 空 | 远程卸载 `0`=连数据删 / 卸载语境（`stop` 输出「已卸载」）/ `fix-deps` 防递归标记 |
| `DSH_HOME` / `DSH_HOME_PARENT` / `DSH_PORT` | 套件数据目录 / 其父目录 / 运行时端口 | dsh 数据与 profile 根（wrapper 尊重已导出值）/ PID 与 `dsh-proxy.log` 落点 / 生成物端口运行时值 |
| `DSH_CLIENT_VERSION` / `DSH_CLIENT_COMMIT_HASH` / `DSH_CLIENT_TITLE` | 打包注入 | 名牌小字版本与标题 |
| `DSH_SLIM_SKIP_NATIVE` / `DSH_REPAIR_DRY_RUN` | `0` / `0` | 跳过 native 构建（留档脚本用）/ 守护只打印将停的 PID 不真停 |
| `DS_FETCH_TOKEN` / `DS_FETCH_GIT_URL` / `DSH_GIT_MIRROR_PREFIX` | 空 / 官方仓库 / 空（关） | 拉源码 token（等价 `--token`）/ git 远端覆盖 / GitHub 镜像前缀（如 `https://gh-proxy.com/`） |
| `DSH_ALLOW_PRERELEASE` | 空（关） | `1` = 拉源码时把 alpha/beta 开发版 tag 也纳入候选；默认只认**正式版**（rc 及以上，排除 alpha/beta） |
| `NPM_REGISTRY_CANDIDATES` | 内置清单 | npm 源并发探测候选数组 |
| `ARCH` / `BUILD_ARCH` | `x86_64` | 目标架构覆盖（`x86_64` / `arm64`） |
| `APP_NAME` `D_SRC` `D_BUILD` `D_STAGING` `D_REL` `D_SCRIPTS` `D_ASSETS` | 读 config | 名称与工作区目录临时覆盖 |
| `vars.BUILD_SPK` / `vars.BUILD_FPK` / `vars.PRUNE_OLD_ASSETS`（Actions） | 空 | 前两者 `false` 跳过对应产物；后者 `false` 关闭残留清理 |

## 排查

见 `docs/故障排查指南.md`（症状 → 判据 → 处置）。

## 文档索引

- [使用指南](docs/使用指南.md) — 安装、访问、端口、配置、常用操作
- [构建与发布指南](docs/构建与发布指南.md) — 三脚本流程、CI、看门狗、架构参数
- [打包规范](docs/打包规范.md) — SPK/FPK 结构与打包不变量
- [故障排查指南](docs/故障排查指南.md) — 症状 → 判据 → 处置
- [CHANGELOG](docs/CHANGELOG.md) — 版本记录

## 许可证与来源

本项目采用 **MIT License**，完整文本见 [LICENSE](LICENSE)；来源与派生关系见 [NOTICE](NOTICE)。

| 层级 | 工程 | 版权 |
|---|---|---|
| 智能体框架 | [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) | Copyright (c) 2026 DeepSeek AI |
| NAS 打包工程（直接上游） | [EIGHTfs/DeepSeekHarness-NAS](https://github.com/EIGHTfs/DeepSeekHarness-NAS) | Copyright (c) 2026 EIGHTfs |
| 本仓（自建二开副本） | [NeeYoonc/dsh-nas](https://github.com/NeeYoonc/dsh-nas) | Copyright (c) 2026 NeeYoonc |

> 本仓为第三方自建副本，与 DeepSeek AI、EIGHTfs 均无隶属关系；Release 产物由本仓自行构建，非上游官方发布物。
> 代码中的套件名 `DeepSeekHarness-NAS` 沿用上游，仅为兼容既有安装路径与数据目录。

