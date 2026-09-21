# Codex 模型管理器 · Windows

解压整个文件夹，双击 CodexModelManager.exe。请保留旁边的 _internal 文件夹。
使用源码时可双击 Start.cmd（先安装 requirements.txt 到 .venv）。

## 日常使用

1. 打开“连接与兼容”，确认 Codex config.toml 路径。默认读取当前 Windows 用户的 .codex。
2. 运行位置选 auto；若 Codex 使用 WSL，填写它使用的发行版，如 Ubuntu。
3. 首次启用前，从托盘退出 Codex。选择 A，适用模型点“GPT”“DeepSeek”或“两者”，点击“启动并启用桥”。
4. 提示成功后重新打开 Codex。之后切换 0/A/B/AB 直接点“切换模式”，无需终端或重启。

### WSL CLI

这套桥也可给 WSL 中的 `codex` / `codex exec` 使用。发布版实测
`codex-cli 0.155.1` 通过桥执行命令成功。CLI 和桥必须使用同一个 `CODEX_HOME`，并且
桥要在同一 WSL 发行版的 `127.0.0.1` 上监听；Windows 侧的 loopback 在 NAT 网络下
不能替代 WSL 侧地址。

```bash
export CODEX_HOME=/mnt/c/Users/<你的 Windows 用户名>/.codex
codex --version
codex exec --json --skip-git-repo-check -C "$PWD" \
  '请实际执行 pwd，只执行这一条命令，不要修改文件。'
```

这验证的是桥的协议兼容性，不承诺修复 CLI 自身的工具装配或上下文压缩故障；如果
请求中本来就没有 `exec` 声明，桥不会自动添加工具。需要完全不经过桥的 CLI 会话时，
使用 README 中的 `--profile` 独立配置方案。

已经运行的脚本桥会被识别，保留原备份和恢复地址。改变适用模型需要退出 Codex 后点“重启桥 / 加载更新”。
桥需要 WSL Python 3.11 或更新版本；Windows 程序自带 Python，不需要手动安装 Windows Python。
首次使用 WSL 可能有几秒启动时间。未安装 WSL 的原生 Codex 用户选择 native。

## 关闭与恢复

- 关闭管理器窗口不会停桥。再次打开后可以查看状态并继续管理。
- 要停止桥：先退出 Codex，点击“恢复直连并停止”，然后重开 Codex。
- 电脑重启后桥不会自动开机启动；打开管理器，退出 Codex 后点击“启动并启用桥”即可恢复。
- 桥离线时页面会提示。不要只杀桥进程而保留 Codex 的本地地址。
- 模式 0 仍经过本地桥；“恢复直连”才会撤掉这一跳。

## 模式与模型

- A：工具协议兼容。只转换选中的准确模型名，未选模型原样通过。
- B：实验性的上下文纠偏；不修改本地保存的历史。
- AB：同时使用。0：两者关闭。
- 压缩请求始终原样透传，压缩后的普通工具请求继续按模式处理。
- GPT 与 DeepSeek 共用同一个 provider 时，即使未选模型也依赖桥运行。
- 模型管理页可添加模型、编辑推理档位、同步、备份、预览并应用目录；真实写入仍需兼容性验证。

“刷新状态”读取配置及本地健康状态，不发送模型请求。“导出诊断”不含 Key、配置正文、对话或代码。
认证仍由 Codex 的 keyring 提供；管理器不读取或复制凭据。
兼容桥是可关闭的临时方案，不承诺修复所有工具问题。

## 界面

采用 Sun Valley ttk 浅色主题（MIT），全界面字体统一为微软雅黑。首页保留常用连接操作；配置文件、端口、发行版、精确模型名及请求记录位于“连接设置与诊断”展开区。
设计参考：https://github.com/rdbende/Sun-Valley-ttk-theme 与 https://learn.microsoft.com/en-us/windows/apps/design/app-settings/guidelines-for-app-settings 。
