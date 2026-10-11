# 请求、返回及官方依据

核实日期：2026-10-04。官方服务使用 deepseek-flash 模型别名；该别名当前对应 V4.1 Flash，今后可能变化。脚本同时记录请求模型名和 API 实际返回的模型名，不宣称固定权重版本。

官方入口：

- https://api-docs.deepseek.com/
- https://api-docs.deepseek.com/guides/vision/
- https://api-docs.deepseek.com/guides/thinking_mode/
- https://learn.chatgpt.com/docs/build-skills

## 本工具请求 JSON

| 字段 | 类型与默认值 | 用法 |
| --- | --- | --- |
| images | 必填，1..8 个路径 | 绝对路径或相对请求 JSON 所在目录的路径；中文、空格支持 |
| question | 必填，非空字符串 | 当前具体观察目标 |
| context | 可选字符串 | 先前证据、疑点、比较要求；当作待核查背景 |
| crop | 可选对象 | image 为从 1 开始的原图序号，box 为归一化矩形 |
| reasoning_effort | low/high/max，默认 high | 官方思考强度枚举 |
| max_tokens | 256..65536，默认 16384 | 本工具输出预算，包含推理与回答；服务仍可能按其规则拒绝 |
| max_side | 64..8192，默认 8192 | 发送前最长边上限；缩小会在结果中记录 |
| timeout_seconds | 10..1800，默认 600 | 总等待上限；单次网络读取最多等 60 秒 |

这些本工具预算不是服务能力上限。较长推理增加延迟和费用，不能保证更准确。

脚本将图片按 EXIF 显示方向规范化、转换为 PNG 后发送；原文件不改动。支持静态 JPEG、PNG、GIF、WebP，动画要求先明确选取静态帧。原图本工具上限 32 MiB；规范化后的单图不超过 32 MiB，序列化请求不超过官方 48 MiB。只支持本地文件，不自动访问未知图片 URL，不走持久 Files API 上传。

每次调用均重新发送所需原图；context 不自动从其他文件载入。全图和裁剪图以独立 image_id 标注。脚本在 PNG 中不保留原文件的 EXIF 元数据。detail=original 仍受服务端视觉处理分辨率约束，不能据此承诺逐像素无损推理。

## 坐标映射

crop.box 的四个值为原图经 EXIF 校正后的 [left,top,right,bottom] / [width,height,width,height]。

实际裁剪用左上向下取整、右下向上取整，记录在 images[].crop_box_pixels 中。

若裁剪返回归一化点 (u,v)，原图坐标为：

~~~text
x = crop_left + u * (crop_right - crop_left)
y = crop_top  + v * (crop_bottom - crop_top)
~~~

使用实际 crop_box_pixels 做映射；裁剪后的可视定位仍是模型估计，不是检测器校准输出。

## API 调用方式

固定 POST https://api.deepseek.com/chat/completions，密钥仅放 Authorization 请求头。不允许重定向到其他地址。

使用 user 消息的 image_url/data URL 块；system 消息只含说明。thinking.type=enabled，stream=true。逐块独立拼接 delta.reasoning_content 与 delta.content，同一个块中两个字段同时出现也保留。记录最后的 usage、id、model、finish_reason。

无 tool 参数的 Flash 请求不需要回传它之前的 reasoning_content；本脚本每轮独立请求。Pro 在 Codex 中的工具调用对话由原有接入维护，不由本技能重写。

## 输出

observe 把完整收到的文字保存在结果 JSON，只向终端打印索引。不自动把长 CoT 当成最终用户答案。主要字段：

- status: complete / incomplete / dry_run。
- content: API 返回的回答字符串，不改写。
- reasoning_content: API 返回的推理字符串，不改写；字段不存在则为 null。
- analysis: 尝试将 content 解析为对象，仅便于读取；不验证内容事实，原文仍保留。
- reasoning_field_seen: 实际是否遇到 API 推理字段。
- transport_complete: 是否收到 SSE [DONE]。
- response_complete: 收到结束标记、finish_reason=stop、有 content 且无传输错误。
- images: 图片标签、原图路径与哈希、规范化后尺寸、真实裁剪范围、发送图像哈希。
- warnings/error: 缺失、截断、格式异常、HTTP 或网络错误。

complete 只表示传输/生成正常结束，不表示视觉结论正确或全部内部计算已公开。

read 子命令按字符偏移分页；next_offset=null 才表示该字段读完。终端自己的显示限制若仍发生截断，应降低 --limit，不应宣称已经读全。

途中断流或输出预算耗尽会保存已收部分并返回退出码 3；不自动付费重试，不自动补写缺失内容，不把多个独立生成拼接成一条完整推理。
