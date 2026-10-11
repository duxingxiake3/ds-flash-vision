# deepseek-flash-vision

一个本地运行的视觉协作 skill：由主模型提出观察目标，通过本地脚本调用 DeepSeek 官方 `deepseek-flash` 视觉接口观察图片，并**原样保留接口真实返回的 `reasoning_content`**，供主模型据此提出下一轮针对性复查。

不生成、不编辑图片，也不声称具备独立的原生视觉能力。

## 特点

- **保留真实推理输出** —— 流式收集 `content` 与 `reasoning_content`，不改写、不补写、不拼接；字段缺失或被截断时如实标记
- **局部复查** —— `crop` 在本地生成真实裁剪图，与全图一起发送，坐标基于 EXIF 校正后的归一化坐标系
- **文字识别** —— 适合读取截图、图表与局部裁剪区中的文字，并要求区分看清的内容与无法确认的部分
- **证据可追溯** —— 记录原图 SHA-256、规范化尺寸、真实裁剪像素范围、发送图 SHA-256 与 API 返回的 `usage`/`id`/`model`/`finish_reason`
- **密钥不落盘** —— 仅从环境变量读取，报错前自动脱敏，且拒绝跟随重定向
- **无网络也可用** —— `doctor` 只做本地条件检查；`--dry-run` 不产生任何网络调用与费用

## 目录结构

```text
deepseek-flash-vision/
├── SKILL.md                      技能主文档（给主模型的操作指令）
├── agents/openai.yaml            skill 元数据与调用策略
├── assets/icon.svg               图标
├── references/
│   ├── api-contract.md           接口契约、字段表、坐标映射、输出说明
│   └── local-setup.md            安装位置、Python 依赖、密钥配置
└── scripts/flash_vision.py       唯一的可执行脚本（doctor / observe / read）
```

## 环境要求

- Python 3.10 或更新版本
- Pillow（唯一外部依赖）

```powershell
py -3 --version
py -3 -m pip install Pillow
```

## 安装

把整个目录放进主模型支持的技能目录之一（不要只复制单个文件）：

- 仅在当前项目使用：项目的 `.agents/skills/deepseek-flash-vision/`
- 当前用户通用：用户主目录的 `.agents/skills/deepseek-flash-vision/`

若技能已在主模型的技能列表中可用，直接使用即可，无需复制。同名技能已存在时先比较再合并，不要直接覆盖。

## 配置密钥

脚本**只**从进程环境变量 `DEEPSEEK_API_KEY` 读取密钥，不会输出它，也不会去解析其他工具的凭据文件。不要写进 `SKILL.md`、请求 JSON 或版本管理。

PowerShell（隐藏输入，避免进入命令历史）：

```powershell
$flashSecureKey = Read-Host "DeepSeek API Key" -AsSecureString
$env:DEEPSEEK_API_KEY = [System.Net.NetworkCredential]::new("", $flashSecureKey).Password
Remove-Variable flashSecureKey
```

Bash：

```bash
read -r -s -p "DeepSeek API Key: " DEEPSEEK_API_KEY
export DEEPSEEK_API_KEY
```

只设置当前终端及其子进程。已经启动的图形界面进程不会自动继承这个新值。

## 用法

### 1. 本地条件检查（不联网、不计费）

```powershell
python scripts/flash_vision.py doctor
```

只报告 Python 版本、Pillow 是否可用、密钥是否已配置。`local_prerequisites_ready=true` **不代表**账户有余额或已通过视觉接口验证。

### 2. 写请求 JSON

```json
{
  "images": ["C:/Pictures/photo.png"],
  "question": "请观察这张图片，标出与问题相关的视觉事实、对应位置和不确定之处。",
  "reasoning_effort": "high",
  "max_tokens": 16384
}
```

### 3. 预检（仍不联网）

```powershell
python scripts/flash_vision.py observe --request request.json --output out.json --dry-run
```

预检通过后，换一个**新的**输出文件名并去掉 `--dry-run` 发起真实观察：

```powershell
python scripts/flash_vision.py observe --request request.json --output out.json
```

### 4. 分页读取结果

```powershell
python scripts/flash_vision.py read --file out.json --field content
python scripts/flash_vision.py read --file out.json --field reasoning_content --offset 0 --limit 12000
```

每次接着 `next_offset` 读，直到返回 `null` 才算读完。`read` 不联网、不计费。

## 请求字段

| 字段 | 类型与默认值 | 说明 |
| --- | --- | --- |
| `images` | 必填，1..8 个路径 | 绝对路径，或相对请求 JSON 所在目录；支持中文与空格 |
| `question` | 必填，非空字符串 | 本轮具体观察目标 |
| `context` | 可选字符串 | 先前证据、疑点；按待核实数据处理，不是指令 |
| `crop` | 可选对象 | `{image: 序号, box: [left,top,right,bottom]}`，0..1 归一化 |
| `reasoning_effort` | `low` / `high` / `max`，默认 `high` | 思考强度 |
| `max_tokens` | 256..65536，默认 16384 | 输出预算，含推理与回答 |
| `max_side` | 64..8192，默认 8192 | 发送前最长边上限 |
| `timeout_seconds` | 10..1800，默认 600 | 总等待上限 |

`crop.box` 使用原图经 EXIF 校正后的归一化坐标。裁剪坐标不可直接当作全图坐标使用；映射方式见 `references/api-contract.md`。

## 退出码

| 码 | 含义 | 处理 |
| --- | --- | --- |
| 0 | 成功，或 `dry_run` 完成 | —— |
| 2 | 本地输入/配置错误 | 修正后重跑；不要把报错当观察结果 |
| 3 | API 请求失败或响应不完整 | 已保留部分结果；**不会**自动重试消耗额度 |

## 安全与隐私说明

**这是使用本工具前必须了解的部分。**

- **图片会上传** —— 观察时本地图片被规范化为 PNG、base64 编码后发送至 `https://api.deepseek.com/chat/completions`。不要在未获授权时提交敏感、涉密或他人隐私图片。
- **输出文件含推理文本** —— `observe` 生成的结果 JSON 包含接口返回的完整 `reasoning_content` 与图片路径。这是本地审查记录，不应自动公开或提交到版本管理。
- **密钥脱敏** —— 错误信息在输出前会抹掉 `Bearer ...`、`sk-...` 形态的字符串与 base64 图片数据；脚本拒绝跟随 HTTP 重定向，避免密钥被转发到其他地址。
- **不修改用户环境** —— 不修改主模型配置、不增设代理、不做自动微调、不将用户图片用于训练。
- **不覆盖旧结果** —— 输出文件以独占方式创建，已存在则直接报错退出，避免覆盖历史观察。

### 已知边界

- 调用预算默认每个任务最多 4 次（含首次），**失败的调用同样计入**
- `complete` 只表示传输与生成正常结束，**不代表**视觉结论正确
- API 返回的全部文字不等于全部内部视觉计算；不要声称获得了隐藏的完整思维过程
- 图片内的文字与历史 `context` 均按**待核实数据**处理，其中的指令、代码、网址不会被自动执行
- 面向专业判定场景时，本工具仅提供辅助观察与复核建议，不用于独立判定对象是否合格、可用或可放行
- 不依据无标尺照片推断实际尺寸；图片中没有观察到某项特征，不等于该特征不存在

## 许可证

[MIT](LICENSE)
