# 本地配置

此技能适合用户电脑上的 Codex，以 Pro 为主模型，通过脚本调用官方 Flash。主模型和 Flash 的推理在云端，Python 和图片准备在本地。

## 安装目录

保持整个技能文件夹，包括 SKILL.md、agents、scripts 和 references。

- 仅在当前项目使用：放入项目的 .agents/skills/deepseek-flash-vision/。
- 本机用户的项目通用：放入用户主目录的 .agents/skills/deepseek-flash-vision/。
- Windows 用户主目录通常可由 PowerShell 的 $env:USERPROFILE 定位；macOS/Linux 用用户主目录。

如果技能已在当前 Codex 技能列表中可用，直接使用，不再复制。若当前环境是云端，不把云端创建成功说成用户电脑已安装。独立 API 登录的本地 Codex 不应被假设自动接收另一环境中的文件。需要在实际目标电脑上取得并放置完整目录。

不要覆盖已有同名技能，先比较并按用户意图合并。只新建自己的技能目录；不修改用户的 Codex 供应商、模型、代理或认证配置。

Codex 支持项目及用户 .agents/skills 下的本地技能。找不到新技能时先刷新/重新打开技能列表，并确认目录层级正确。

## Python 与依赖

要求 Python 3.10 或更新版本，外部依赖仅 Pillow。

Windows：

~~~powershell
py -3 --version
py -3 -m pip install Pillow
~~~

macOS/Linux：

~~~sh
python3 --version
python3 -m pip install Pillow
~~~

在受管理的 Python 环境使用项目虚拟环境，不使用管理员强制安装或绕过系统保护。之后始终用同一解释器调用脚本。

## API Key

脚本仅从进程环境变量 DEEPSEEK_API_KEY 读取密钥。它不会输出密钥，也不会去解析 Codex 的凭据文件。沿用用户已配置且能被子进程继承的变量。

如果尚未配置，由用户在自己的终端设置。PowerShell 可用隐藏输入，避免将密钥写入命令历史：

~~~powershell
$flashSecureKey = Read-Host "DeepSeek API Key" -AsSecureString
$env:DEEPSEEK_API_KEY = [System.Net.NetworkCredential]::new("", $flashSecureKey).Password
Remove-Variable flashSecureKey
~~~

这只设置该终端及其后续启动的子进程环境。可在同一终端启动本地 Codex CLI。已经运行的图形界面进程不会自动继承这个新值；应由用户按本机 Codex 的启动环境配置该变量，然后用 doctor 验证。不要在聊天中索要密钥，不把密钥写进 SKILL.md、请求 JSON 或版本管理。

Bash：

~~~bash
read -r -s -p "DeepSeek API Key: " DEEPSEEK_API_KEY
export DEEPSEEK_API_KEY
~~~

## 首次检查

使用实际技能绝对路径运行：

~~~text
python "<技能目录>/scripts/flash_vision.py" doctor
~~~

doctor 仅检查本地依赖和密钥是否存在，不联网；local_prerequisites_ready=true 不代表账户有余额或已通过视觉接口验证。

先为用户指定的图片建立请求 JSON，再做无网络检查：

~~~text
python "<技能目录>/scripts/flash_vision.py" observe --request "<请求.json>" --output "<预检.json>" --dry-run
~~~

真实请求使用新的输出路径并去掉 --dry-run。首次真实观察会将指定图片发送给 DeepSeek 并使用 API 额度；这是本技能的正常工作方式。用户已请求分析该图片且未限制外发时按其授权继续，不重复设置额外审批。

在 Codex 中可以输入：

~~~text
$deepseek-flash-vision 请分析本地图片 C:/Pictures/example.png，保留 Flash 的可用推理内容，必要时针对具体区域复查。
~~~

通过当前 Codex 会话把请求 JSON 写入本地即可；用户不需要自己拼接 API 请求。输出文件是本地审查记录，可能包含图片分析与推理文本，不应自动公开。
