# Codex Model Manager — Windows 移植版

**新增 Windows 图形连接管理。** 便携包解压后双击 `CodexModelManager.exe`，源码可双击 `Start.cmd`。详见 [QUICKSTART.md](QUICKSTART.md)。

首页支持启动桥、0/A/B/AB 切换、GPT / DeepSeek / 两者作用范围、重启桥、恢复直连和诊断导出，自动选择 Windows / WSL 后端。无需 WSL 终端；打开界面不修改 Codex 配置。

**当前版本：0.1.0 release candidate**。Windows 目标环境的完整测试已通过；发布前只需在
目标 Windows checkout 中按下方命令复跑一次，并确认桥保持默认关闭。

发布命令和安全检查见 [RELEASE_CHECKLIST.md](RELEASE_CHECKLIST.md)。

在 Windows 上查看、编辑、合并、备份 Codex 官方模型与自定义模型的最小编译版。
这是面向 Windows 的独立实现，基于 [gaofeng21cn/opl-codex-models](https://github.com/gaofeng21cn/opl-codex-models) 的模型目录能力扩展。许可证与改动来源见 [SOURCE_NOTICE.md](SOURCE_NOTICE.md)。

> 默认运行在 **预览 / 沙箱模式**：`sync` 只在隔离的临时 `CODEX_HOME` 里读取官方目录并写到**独立应用沙箱**
> 里的本地合并目录（默认 `%LOCALAPPDATA%\CodexModelManager`），**不会**改动你的真实 `~/.codex/config.toml`。
> 真实写入必须通过显式的 `apply` 操作（目标路径只取自配置 `codexConfigPath` 或 `--codex-config`，绝不从环境推导；
> 先展示安全的 `model_catalog_json` 预览，再备份、原子写入）。
> 差异/预览从不回显令牌或密钥，也不会带出配置里其他无关字段。
>
> **真实 WSL 后端可用**：经 `probe --verify` 产出绑定运行时身份的兼容性证据后，`apply` 在真实模式下由"恒定拒绝"
> 改为**证据门控**——证据有效才允许写入。证据探测与真实写入均在隔离的临时 `CODEX_HOME` 内完成，
> 不修改用户真实 `config.toml`/`auth.json`，不安装新 Codex，不发模型请求。详见"验证情况"与 [HANDOFF.md](HANDOFF.md) 第 16 节。

## 界面预览

连接与桥控制：

![连接与兼容界面](docs/screenshots/connection-bridge.png)

模型目录接管与编辑：

![模型管理界面](docs/screenshots/model-management.png)

## 技术栈

- Python 3.11+（开发/验证环境为 3.12）
- tkinter（内置图形界面）
- tomlkit（保留注释的 TOML 编辑）
- 无第三方运行时依赖（子进程为参数数组 + 超时 + 退出码检查，不用 shell 拼接）

## 全新虚拟环境运行（Windows）

```powershell
cd opl-codex-models
py -3.12 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -U pip
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

### 1) 先用离线样例跑通（模拟验证，不需要真实 Codex）

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\offline_demo.ps1
```

该脚本生成一个 mock `codex` 运行时 + 隔离沙箱目录，依次演示：
`probe → sync → list → add → reasoning → 二次 sync(no_change) → backup → apply --dry-run`。
所有结果都是**模拟验证**，已明确标识。

### 2) 启动 GUI（默认真实管理界面；模拟界面追加 -Demo）

```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\launch_gui.ps1
```

### 3) 运行测试

```powershell
$env:PYTHONPATH = (Join-Path (Get-Location) "src")
.\.venv\Scripts\python.exe -m pytest .\tests -q
```

## 日常用法（CLI）

```powershell
$env:PYTHONPATH = (Join-Path (Get-Location) "src")
$py = ".\\.venv\\Scripts\\python.exe"

# 探测本机真实运行时（Windows 原生 PE 优先，兼列出 WSL ELF 二进制）
& $py -m codex_model_manager --config <cfg> probe --wsl

# 兼容性探测 + 证据验证（真实 WSL 后端：在隔离 CODEX_HOME 内写带唯一 slug 的
# model_catalog_json 再由运行时读回，产出绑定运行时身份的证据）
& $py -m codex_model_manager --config <cfg> probe --verify

# 只读同步（隔离 CODEX_HOME，不写真实 config）
& $py -m codex_model_manager --config <cfg> sync --append-log

# 列出模型 / 新增自定义模型 / 编辑推理档位 / 备份恢复
& $py -m codex_model_manager --config <cfg> list
& $py -m codex_model_manager --config <cfg> add my-model --template gpt-6-astra --context-window 131072
& $py -m codex_model_manager --config <cfg> reasoning my-model --efforts low,high,max --default high
& $py -m codex_model_manager --config <cfg> backup
& $py -m codex_model_manager --config <cfg> restore <backup.bak>

# 显式应用到真实 Codex config（先看差异，dry-run 不写）
& $py -m codex_model_manager --config <cfg> apply --diff --dry-run
& $py -m codex_model_manager --config <cfg> apply   # 确认后执行（先备份；真实模式需有效证据）

# 撤销上一次 apply（还原 config.toml，检测冲突）
& $py -m codex_model_manager --config <cfg> undo
```

> `<cfg>`：应用自身的 JSON 配置。不传时默认
> `%LOCALAPPDATA%\CodexModelManager\config.json`。模型源/合并目录默认在应用沙箱内，
> 读取官方目录时才会临时隔离 `CODEX_HOME`。Windows 与 WSL 的运行时被明确区分。
> `apply` 的目标 config.toml 只取自 `codexConfigPath` 或 `--codex-config`，绝不会从环境推导。

## 接管已有目录（生效目录 ↔ 待应用目录）

Codex 实际读取的是 config.toml 里 `model_catalog_json` 指向的那个文件，它通常在手写目录里、
位于管理器沙箱之外。接管流程把这份现有目录纳入管理，且不会悄悄删掉你已有的模型：

```powershell
# 1) 把 Codex 当前引用的目录复制成管理器的“待应用目录”（保留全部未知字段；
#    默认从 codexConfigPath 的 model_catalog_json 读取，也可用 --active 显式指定）
& $py -m codex_model_manager --config <cfg> takeover-import

# 2) 编辑副本（默认编辑待应用目录；只改给出的字段，未知字段与其他模型不动）
& $py -m codex_model_manager --config <cfg> edit-model gpt-6-astra --context-window 262144 --max-context-window 262144
& $py -m codex_model_manager --config <cfg> edit-model gpt-6-astra --efforts low,high,max --default max

# 3) 只读查看差异：新增 / 修改 / 删除，删除会被单独标出
& $py -m codex_model_manager --config <cfg> takeover-diff

# 4) 写回生效目录（先显示差异并需确认；删除已有模型要 --confirm-removals）
& $py -m codex_model_manager --config <cfg> takeover-apply
& $py -m codex_model_manager --config <cfg> takeover-apply --confirm-removals

# 5) 撤销写回（仅在目录仍等于上次写回值时才恢复，否则报冲突）
& $py -m codex_model_manager --config <cfg> takeover-undo
```

同一套流程在 GUI 的「模型管理」页上：顶部固定显示`生效配置引用目录（Codex 实际读取）`与
`待应用目录（管理器编辑，尚未生效）`，并列出「从现有目录导入副本…」「差异预览（只读）」
「写回生效目录…」「撤销写回」；把视图切到`待应用目录（接管）`后，列表里的每个模型都可编辑
（推理档位、默认档位、上下文、名称/描述），也可新增。

安全语义：写入前自动备份原文件、原子替换；待应用目录结构非法则拒绝；生效目录在导入后被外部
修改则拒绝并提示重新导入；删除已有模型必须显式确认；演示模式沿用同一套沙箱限制，真实模式仍
要求绑定当前运行时的有效兼容性证据。只读写模型目录 JSON，不改 config.toml、不碰凭据。

## 本地桥（可选，默认关闭）

**为什么需要它**：Codex 把 `exec` 工具作为 Responses 的 `custom` 工具发送
（`use_responses_lite=true` 时放在 `input[].additional_tools`，`=false` 时放在顶层
`tools[]`）。只接受普通 function 工具的中转会直接拒绝：

```
Unsupported custom tool: 'exec'
```

本地桥在中间做协议转换，把 `custom` 降级为 `function`（**名字保持 `exec`，不加
`functions__` 前缀**），再把上游返回的 `function_call` 还原成 Codex 期望的
`custom_tool_call`，`call_id` 原样保留。

### CLI 也可以直接使用本地桥

发布版桥同时支持 WSL 中运行的 `codex` / `codex exec`，不要求通过 GUI 发起对话。
实测组合为 `codex-cli 0.155.1`、Responses provider、同一份 `CODEX_HOME`，CLI
通过桥执行了真实的 `pwd`，收到 `exit_code=0`。因此遇到 CLI 侧的
`Unsupported custom tool: 'exec'` 时，可以先用同一条桥验证协议兼容性。

使用时必须满足两个路径条件：

1. CLI 和桥使用同一个 `CODEX_HOME` / `config.toml`，否则 CLI 可能仍然直连旧地址；
2. CLI 在 WSL 中运行时，桥也必须在**同一个 WSL 发行版**内监听 `127.0.0.1`。Windows
   侧的 `127.0.0.1` 在 NAT 网络下不是 WSL 后端的同一个 loopback。

最短验证流程如下。先用 GUI 的“启动并启用桥”，或按本节 CLI 命令启用桥；然后在
WSL 终端执行：

```bash
export CODEX_HOME=/mnt/c/Users/<你的 Windows 用户名>/.codex
codex --version
codex exec --json --skip-git-repo-check -C "$PWD" \
  '请实际执行 pwd，只执行这一条命令，不要修改文件。'
```

若使用 `--only-model deepseek-v4.1-flash`，只有该模型的报文会转换，其他模型原样
透传；但 provider 的 `base_url` 仍指向桥，所以桥停止时 CLI 的其他模型也会暂时失败。
若要让 CLI 真正完全绕开桥，请使用下方的 `--profile` 独立配置方案。

这项验证证明的是“CLI → 桥 → 中转”的协议链可用，不保证 Codex CLI 自身的长期工具
装配或上下文压缩问题已经修复。如果 CLI 在发请求前就没有声明 `exec`，桥无法凭空补回
工具；可用 `bridge start --record <脱敏日志>` 或 GUI 的诊断导出继续定位。

> **默认关闭，不开机自启动。** GUI 首页管理后台桥，健康检查后切换地址，恢复直连后停止服务。关闭窗口不停桥。以下 CLI 命令仅供开发使用，日常操作不需要终端。

```powershell
# 1) 确认默认是关闭的
& $py -m codex_model_manager --config <cfg> bridge status

# 2) 把当前 provider 指向本地桥（只改 base_url 一项；保注释，写前自动备份）
& $py -m codex_model_manager --config <cfg> bridge enable
#   可选：--host 127.0.0.1 --port 8787 --provider <名字> --upstream <真实中转>

# 3) 另开终端启动桥进程（前台运行，Ctrl+C 停止）
& $py -m codex_model_manager bridge start

# 4) 关闭开关，恢复原来的 base_url（检测冲突；当前值被改过就不动手）
& $py -m codex_model_manager --config <cfg> bridge disable
```

GUI 首页对应「**启动并启用桥**」/「**恢复直连并停止**」两个按钮：会先弹出 `旧地址 -> 新地址`
的差异确认，只改这一个键，取消则不写任何内容。

**安全约束（已由测试锁定）**

- 只允许绑定 loopback（`127.0.0.1` / `localhost` / `::1`）。该校验写在 `BridgeServer`
  数据模型内部，因此 CLI、GUI、脚本、直接构造对象**都绕不过**；写 `0.0.0.0` 会在任何
  "已启动"提示之前就被拒绝。
- **不保存 API Key**：桥只透传 Codex 已经带来的 `Authorization` 头，没有凭据落盘。
- 与 `apply` 遵守同一条 demo 沙箱规则：演示模式下目标越界（`..`/符号链接/沙箱外路径）
  一律拒绝写入。
- 上游报错原样透传（例如 400 `Unsupported custom tool`），**不会伪造成功**。
- **转换只依据显式注册映射**，不做字符串猜测：
  - `functions__exec` 不再被当成「协议异常一律拒绝」。它是命名空间拼名的合法产物——
    `functions.exec` 是 Codex 内部记法，`exec` 才是可调用名——桥按注册表精确还原为
    `custom_tool_call(name="exec")`；
  - 无注册（`undeclared_in_this_conversation`）或注册歧义（同一拼写被两条注册认领）
    一律**拒绝转换并记账**，在 stderr 报告。
- **映射按会话隔离**：注册表以 `previous_response_id` / 客户端会话字段为键，LRU 512 + 空闲
  3600s 过期；没有任何全局名称集合，因此两个会话使用同名但不同类型的工具不会互相污染。
  历史项会被降级，但**不写入注册表**——历史是记录，不是当前轮次的工具授权。
- **SSE 帧格式按真实客户端的要求发**：`response.created` → `response.in_progress` →
  每个 item 的 `output_item.added` + `output_item.done` → `response.completed`，每个事件带
  递增 `sequence_number`。这个 Codex 客户端**不处理 `output_item.added`**（该字面量在其
  二进制里 0 命中），只从 `output_item.done` 组装 item；只发 `added` 会让整轮被静默丢弃。

**凭据**：桥自身**不保存、不注入**凭据，只透传 Codex 带来的 `Authorization` 头。
离线/取证脚本取值一律走内存通道（`scripts/l2_credentials.py`：`env` / `secret-tool` /
`stdin`），**刻意没有把 Key 当命令行参数的选项**，且不关闭 TLS 校验。

> **`secret-tool lookup` 会把秘密写到它自己的 stdout。** 它不像某些工具那样走专用
> 句柄，行为等同于 `cat`。因此规则是硬性的：那条管道必须**由程序捕获**
> （`subprocess.run(..., capture_output=True)`），绝不允许交给终端、日志、取证文件，
> 也绝不允许出现在**异常正文**里——本模块的错误分支只报退出码和截断后的 account 摘要，
> 连子进程的 stderr 都不引用，因为"把子进程输出贴出来看看"正是密钥泄漏进会话记录的方式。
> 要诊断就用 `describe()`（只回长度类别）或先过 `redact()`。
>
> **已暴露过的旧 Key 不再读取、不再打印、不再比对。** 本模块刻意没有"跟旧值比一比"的
> 代码路径：泄漏出去的值按废弃处理，轮换走安全入口。`tests/test_credentials.py`
> 把上述口径钉成可执行断言（含"仓库内不得存在疑似真实凭据"的扫描）。

**模型范围（`--only-model`）与"开关只影响 DeepSeek"这句话**

`base_url` 是 **provider 级**的设置，而这台客户端的模型目录、app-server 的 `Model`
类型、桌面端的模型选择器都**没有 provider 维度**（模型选择器存的是
`{model, reasoningEffort, serviceTier}`），`[profiles.*]` 又已是 legacy。所以
**不能按模型选择 provider**——把 provider 指向桥，就等于把所有模型都送上桥。

`bridge start --only-model deepseek-v4.1-flash` 能做的是把影响面收窄，而不是消除：

- 范围内的模型照旧做协议转换；
- 范围外的模型**请求与响应字节不变**地直通（不剥离 `additional_tools`、不改 `stream`、
  不重新序列化 JSON、不写取证），因此它们看到的就是直连中转时会看到的东西；
- **但桥仍然挡在所有模型前面**：桥没起来时，范围外的模型一样会失败。

所以正确的说法是"只有 DeepSeek 的**报文**会被改写"，而不是"开关只影响 DeepSeek"。
需要**真隔离**（GPT 连一跳都不加）时，见
[desktop-enable.md](desktop-enable.md) 的方案 B：给 CLI 用一份独立的
`<name>.config.toml` + `codex --profile`，全局 provider 不动。

**取证开关（默认关闭）**

`--record` 与 `--capture` 都是**可选、默认不启用**的，常规启动命令
（`bridge start` / `bridge_l2_serve.py --upstream …`）都不带它们。

| 开关 | 写什么 | 敏感性 |
|---|---|---|
| `--record` | 每请求一行**脱敏元数据**：工具名/类型、哈希后的 call id、长度、诊断 | 无提示词、无参数、无请求头、无凭据 |
| `--capture` | 每个请求一份 `upstream-N.json`（上游原始响应体）与 `client-N.sse`（实际发给客户端的字节），加 `index.jsonl` | **模型输出，可能敏感**。只写盘，**本程序从不打印、从不汇总进日志、也从不自动删除** |

范围外的模型（`--only-model` 之外的）**不会被捕获**，`--capture` 只覆盖真正经桥转换的请求。
现存取证文件的位置与体量见 [desktop-enable.md](desktop-enable.md) 的"现有取证文件"一节。

**验证（不碰真实配置）**

```powershell
# L1 端到端：内置假中转，零外部依赖。任意断言失败即非 0 退出
& $py scripts/bridge_l1_e2e.py

# L2 协议闭环探针：合成白名单工具 probe_echo。--fake 为离线自检
& $py scripts/bridge_l2_probe.py --fake --path both
& $py scripts/bridge_l2_probe.py --upstream https://<真实中转> --credential secret-tool --record <out.jsonl>

# SSE 帧格式离线回放：真实客户端 + 零模型调用
& $py scripts/bridge_replay.py --capture <capture_dir> --port 8798
```

L1 会断言：`additional_tools` 信封被剥离、`exec` 降为 `function` 且名字不变、上游收到
`stream=false`、客户端拿回 SSE 里的 `custom_tool_call`、**第二轮**只带历史（无工具声明）
时历史被降级为 `function_call`/`function_call_output` 且 `call_id` 字节级配对。

**真机结果（第八轮，详见 HANDOFF 第 18 节）**

- **真实中转协议闭环**：`deepseek-v4.1-flash` 经本地桥，三条场景（`additional_tools`×low、
  顶层 `tools[]`×low、`additional_tools`×high）**全部闭环**——模型返回结构化调用、桥还原
  类型/名称/参数/`call_id`、测试程序回结果、模型下一轮据结果作答。证据
  `integration-evidence/l2-protocol-loop.json`。
- **真实 Codex 隔离接入**：隔离 `CODEX_HOME`、进程级 `env_key` 注入凭据，四轮
  （列目录 → 写 `hello` → 下轮读回 → 改为 `world` 读回）**全部 exit 0**，桥 8 个请求
  `diagnostics=[]`；真实全局 `config.toml` sha256 前后一致。证据
  `integration-evidence/l2-codex-acceptance.json`。
- **边界**：该 Codex **恒用 `additional_tools`**，顶层 `tools[]` 未被真实客户端触发（其覆盖
  来自协议探针与 L1）；此前记录中的 `high` 仅指第八轮探针；第九轮已有隔离客户端 high 验收，`max` 尚未做桌面验收。

> 离线测试通过只说明**桥自身**的转换正确。真实中转与真实客户端的兼容性已由第八轮的 L2
> 取证确认（`deepseek-v4.1-flash`：协议闭环 + 真实 Codex 四轮验收），步骤与证据见
> [HANDOFF.md](HANDOFF.md) 第 18 节。更换中转或更换 Codex 版本时，应重跑本节脚本——
> 尤其 `bridge_replay.py`，它能用零模型调用验证 SSE 帧格式是否仍被客户端接受。

## 配置与目录

| 用途 | 默认位置 |
|---|---|
| 应用配置 | `%LOCALAPPDATA%\CodexModelManager\config.json` |
| 自定义模型源 | `%LOCALAPPDATA%\CodexModelManager\custom-models.json`（应用沙箱） |
| 合并目录 | `%LOCALAPPDATA%\CodexModelManager\models.json`（应用沙箱） |
| apply 目标 config.toml | 仅由配置 `codexConfigPath` 指定；未设置则不可真实写入 |
| 同步日志 / 错误日志 | `%LOCALAPPDATA%\CodexModelManager\Logs\sync.jsonl` / `sync.error.log` |
| 备份 | `%LOCALAPPDATA%\CodexModelManager\Backups` |

桌面客户端启用本地桥的操作手册（配置差异 / 启动命令 / 关闭与恢复 / 限制）：
[desktop-enable.md](desktop-enable.md)。

GPT 间歇性工具不可用的两个可选实验补丁（协议兼容 / 上下文纠偏）、独立开关与四组比较：
[TOOL_RECOVERY.md](TOOL_RECOVERY.md)。该功能未宣称已根治真实 GPT 故障；原任务历史保持不变。

## 目录结构

```
src/codex_model_manager/
  core/
    errors.py            # 错误类型
    process_runner.py    # 参数数组子进程 + 超时 + 退出码
    runtime.py           # Windows/WSL 运行时探测 + CODEX_HOME
    wsl_adapter.py       # RuntimeTarget 统一适配（native/WSL，参数数组，env 过滤）
    compat_probe.py      # 兼容性探测 + 证据生成/校验（probe_scheme=wsl-adapter-v1）
    catalog_sync.py      # 合并同步（隔离 CODEX_HOME、优先级、保未知字段）
    overrides.py         # 字段级上下文覆盖
    custom_models.py     # 自定义模型新增/编辑
    reasoning.py         # 推理档位
    config_editor.py     # tomlkit 保注释编辑 model_catalog_json / provider base_url
    safe_preview.py      # 写入闸门（apply + 本地桥共用同一套 demo 沙箱规则）
    bridge.py            # 可选本地 Responses 桥（默认关闭）：显式注册映射 + 会话隔离 + SSE 帧
                         #   + 模型范围（--only-model：范围外请求/响应字节级直通）
    backup.py            # 原子写 + 备份/恢复
    app_config.py        # 应用配置
  parser.py              # 模型与日志解析
  services/catalog_data_service.py  # GUI+CLI 共享
  cli.py / __main__.py   # 命令行入口
  gui/app.py             # tkinter 界面
demo/                    # 离线样例：mock codex、样例自定义模型、probe 抓取样本
scripts/                 # offline_demo / setup_demo / launch_gui / gui_smoke
                         # bridge_l1_e2e（L1 端到端）/ bridge_l2_probe（L2 协议闭环探针）
                         # bridge_l2_serve（常驻桥）/ bridge_replay（SSE 帧离线回放）
                         # l2_credentials（内存凭据通道；刻意没有命令行传 Key 的入口）
tests/                   # pytest 行为测试（含 test_compat_probe.py、test_bridge.py、
                         # test_credentials.py：凭据通道的安全口径）
integration-evidence/    # 真实 WSL 接入 + L2 协议闭环 + Codex 隔离验收证据
                         # 真实联调的内部脚本不随发布目录交付；脱敏结果保留在
                         # integration-evidence/，可公开重跑的桥验证脚本在 scripts/。
```

## 验证情况（详见 HANDOFF.md）

- **模拟验证**：与原始 Swift 测试语义对齐的合并/覆盖/自定义/TOML/备份/中文空格路径/子进程失败与超时用例全部通过，另有验收整改的回归用例。当前 Windows 虚拟环境全套 `pytest tests -q -rs` 为 **230 passed，1 skipped**（2026-09-20，本机跳过符号链接权限项）；受限环境若无法创建符号链接，symlink 越界用例会按平台能力跳过。`apply --diff/--dry-run` 严格只读、安全预览不回显密钥、恢复按内容类型校验、`recommended()` 落应用沙箱、演示流程不改真实 CODEX_HOME、probe 只报告已验证能力、无运行时/无显示下 GUI 可安全 import 等均已覆盖；接管流程还覆盖显式目标一致性、任意目录写入拒绝、旧配置迁移、写回前持久化撤销记录与删除确认。**Windows junction 越界已由可运行用例实测拒绝**。mock CLI 全流程跑通。
- **真实 WSL 接入（第六轮）**：已从"离线原型"推进到"真实 WSL 后端可用、用户可显式启用"。实测链路 `Windows → wsl.exe --distribution Ubuntu --exec env CODEX_HOME=<linux> <runtime> ...`（参数数组，不用 `bash -lc`）跑通；运行时身份 `codex-cli 0.155.0-alpha.9`、distro `Ubuntu`、SHA256 `b544b069…d82e`。兼容性探测 `probe --verify` 产生绑定运行时身份的证据（`probe_scheme=wsl-adapter-v1`），含正控制（加载 `model_catalog_json`、`marker_loaded=true`）与负控制（bundled 不含 marker、缺目录退出 1）。`apply` 在真实模式下改为**证据门控**：证据有效才允许写入，不再恒定拒绝；`undo` 可还原上一次 apply 并检测冲突。所有真实写入均在隔离的临时 `CODEX_HOME` 内完成，**不修改用户真实 `config.toml`/`auth.json`，不安装新 Codex，不发模型请求**。证据与可重跑脚本见 [integration-evidence/](integration-evidence/)，详见 [HANDOFF.md](HANDOFF.md) 第 16 节。
- **本地桥（第七轮）**：CLI `bridge enable/start/disable/status` 与 GUI「启用/关闭本地桥」按钮已可用，**默认关闭**。`tests/test_bridge.py` 共 **48 项**（当前 WSL 环境可运行项通过；5 项 GUI 需 Windows/tkinter），含 wire 形状单测、注册映射与歧义拒绝、两会话同名不同类型工具的隔离、跨轮次语义保持、loopback 拒绝、demo 沙箱越界拒绝、SSE 帧格式回归（`output_item.done` + 连续 `sequence_number`）、**模型范围化直通**（范围外模型收发字节逐字节不变、不被捕获、`mode=passthrough` 记账），以及一个**内置假中转的 L1 端到端**用例（不依赖真实 Codex、不写真实配置）。CLI 沙箱冒烟实测走通「关闭 → 启用（改 base_url、留注释、留备份）→ 状态 → 关闭（逐字节还原）」，`0.0.0.0` 在 CLI 与数据模型两层都被拒绝。`gui_smoke.py --bridge` 真实驱动 GUI 方法完成启用/关闭往返。
- **L2 真机联调（第八轮，真跑）**：`deepseek-v4.1-flash` 经本地桥对真实中转，协议闭环三条场景（`additional_tools`×low、顶层 `tools[]`×low、`additional_tools`×high）全部成立；随后用**真实 Codex 0.155.0-alpha.9.2**（隔离 `CODEX_HOME` + 进程级 `env_key` 注入凭据）完成四轮验收（列目录 → 写 `hello` → 下轮读回 → 改 `world` 读回）**全部 exit 0**，桥 8 个请求 `diagnostics=[]`，真实全局 `config.toml` sha256 前后一致。过程中定位并修复一个只在真实客户端暴露的缺陷：该 Codex **不处理 `response.output_item.added`**，桥只发 `added` 导致整轮被静默丢弃。证据：`integration-evidence/l2-protocol-loop.json`、`integration-evidence/l2-codex-acceptance.json`，详见 [HANDOFF.md](HANDOFF.md) 第 18 节。
- **桌面试用与模型范围化（第九轮，真跑）**：新增 `--only-model`，把桥变成**模型范围化**——范围外模型请求/响应**字节级直通**（不改 `stream`、不剥 `additional_tools`、不重排 JSON、不写取证），因为实测确认这台客户端**不能按模型选择 provider**（单 provider + 目录/协议/选择器都没有 provider 维度 + `profiles` 已是 legacy）。用**已轮换的凭据**在隔离 `CODEX_HOME` + 专用目录里跑真实客户端：低档位四轮**同一会话**（列目录 → 写 `hello` → 下轮读回 → 改 `world` 读回）＋ high 档位**独立会话**，**五轮全部 exit 0、所有命令 exit_code=0**，桥 11 个翻译请求 `diagnostics=[]`；`gpt-6-astra` 经同一桥对照 HTTP 200 且桥记录 `mode=passthrough`；真实全局 `config.toml` 在**本轮运行窗口内** sha256 前后一致（窗口之后该文件被并行处理"掉工具"的会话重写过，键集合与 provider/base_url 均未变，详见 [HANDOFF.md](HANDOFF.md) 第 19.6 节）。另用零模型调用的死端探针验证 `codex --profile <name>` 确实提供**独立 provider 层**（基础层未被拨号）。`tests/test_credentials.py` **11 passed**（秘密只进内存、不进 stdout/日志/异常正文；`secret-tool` 的 stdout 必须由程序捕获；仓库全域无疑似真实凭据）。操作手册见 [desktop-enable.md](desktop-enable.md)，证据 `integration-evidence/desktop-trial.json`，详见 [HANDOFF.md](HANDOFF.md) 第 19 节。
- **仍未验证**：顶层 `tools[]` 路径**未经真实 Codex 客户端触发**（该 Codex 恒用 `additional_tools`，其覆盖来自协议探针与 L1）；**GUI 桌面端本身未被驱动**（本轮只跑同一 Codex 运行时 + 隔离 `CODEX_HOME`，桌面上的点击需用户按手册自行完成）；`--only-model` 范围化在真实客户端上只验证了"DeepSeek 翻译 + GPT 直通可用"，**没有**验证"开关打开后 GPT 的可用性不受桥进程影响"（这一点按设计就不成立）；本机仍无 Windows 原生 Codex。
- **真实验证**：在已检查的位置（显式路径、PATH、一个 `~/.codex/bin/wsl/<sub>` 目录）未找到 Windows 原生 Codex（唯一二进制是 `~/.codex/bin/wsl/.../codex`，为 ELF/WSL）。因此没有原生 Windows 真实集成测试；真实集成走 WSL 后端。
- **接口探测**：经 WSL 探测 `codex-cli 0.155.0-alpha.9`，`debug models --bundled` 输出 `{"models":[...]}`。该输出是数据（bundled 目录），不是配置 schema；`probe --verify` 通过在隔离 `CODEX_HOME` 内写入带唯一随机 slug 的 `model_catalog_json` 再由运行时读回，**已构成 `model_catalog_json` 配置读取能力的证据**。
- **GUI**：`gui_smoke.py` 通过 `tk.Tk()` 创建根窗口，**需要显示桌面环境**，不是无头测试；无显示环境下仅验证 GUI 模块可安全 import 且离线/可恢复路径可达。SettingsDialog 可切换 backend/distro/runtime_path/CODEX_HOME/config_path，"验证兼容性"按钮触发 `probe --verify`，"撤销 apply"按钮触发 `undo`，"启用本地桥…/关闭本地桥"按钮触发桥开关（先展示 `旧地址 -> 新地址` 差异再写），状态栏反映证据有效/失效状态。

## DeepSeek V4.1 Flash 推理档位

项目提供的目录条目声明 `low`、`high`、`max` 三档，默认 `high`。官方 [DeepSeek Thinking Mode 文档](https://api-docs.deepseek.com/guides/thinking_mode) 列出这三个 `reasoning_effort` 值；中转是否实际应用请求档位仍需用服务端请求记录验证。需要更新已有显式目录时，可运行：

```bash
python scripts/update_reasoning_levels.py --target <显式 model_catalog_json 路径>
```

## GPT 工具链只读诊断

GPT 偶发“没有 exec”与 DeepSeek 协议转换是两条独立链路。本项目提供 `doctor`，只读取
应用配置、provider 地址、bridge loopback 可达性、显式运行时文件以及当前系统能观察到的
`codex`/`codex-code-mode-host` 进程；它**不能读取桌面当前回合的 tools 清单**，也不会重启、
改配置或读取 API Key。没有观察到 `code-mode-host` 只能记为旁证，不能直接当根因。

```powershell
& $py -m codex_model_manager --config <app-config.json> doctor `
  --codex-config <显式 config.toml> --observe-log <obs.jsonl> --json
```

诊断输出中的 `tool_registry=unobservable` 是诚实边界：仍需在桌面新 GPT 回合实际执行一条
命令来确认工具入口。观察代理日志只汇总客户端已经发出的工具元数据；若出现
`observation=request_missing_tools`，说明工具在发出请求前就缺失，桥无法补回。`processes=host_not_observed`
或 `bridge=unreachable` 只提示需要关注，不会自动修复或覆盖用户配置。

脚本会先备份目标，只修改 `deepseek-v4.1-flash` 条目的推理字段。

## 后续范围

完整自动同步（计划任务/常驻）、安装向导及自动更新（已有 PyInstaller 便携包）、真实 Windows 原生 Codex 集成（当前真实集成走 WSL 后端）、真实桌面客户端的人工回归，以及**客户端侧按线程选 provider**（协议已支持 `ThreadStartParams.modelProvider`，缺的是桌面端入口——那属于客户端改动）。这些不阻塞当前便携版发布。

## 图形连接管理验证（2026-09-20）

230 passed, 1 skipped（Windows 符号链接权限）。实际 tkinter 窗口测试覆盖启动、状态重连、四种模式、两模型范围、重启、逐字节恢复、模型页和接管流程；均使用临时配置与假上游。便携 EXE 的 native worker 通信、打包附带 WSL worker 从 Windows 查询当前桥均通过，真实配置哈希不变。

本轮未替用户重启真实桥或发真实模型请求，不承诺永久修复 GPT 工具丢失。构建方式：Windows Python 安装 PyInstaller 后执行 `python scripts/build_windows.py`；产物 `dist/CodexModelManager-Windows-portable.zip`。WSL 后端需要 Python 3.11+。构建拒绝覆盖含 user-data 的便携目录。
