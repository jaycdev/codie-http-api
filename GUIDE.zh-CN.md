# Codie 1.6.0 HTTP API 内测接入指南

适用范围：运行 1.6.0 的 Codie 设备，供内测人员在局域网内通过脚本、桌面或移动应用接入。文档按 master 分支源码整理，不代表所有设备已收到该版本，也不代表全部接口已经完成设备实测。

## 文档包怎么用

解压 `Codie-HTTP-API-1.6.0.zip`，双击 `index.html`，即可离线搜索和查看全部公开接口的参数、请求结构、响应结构及功能权限；实际操作数和路径数见 `manifest.json`。无需启动服务或访问设备上的实验室页面。

| 文件 | 用途 |
| --- | --- |
| `index.html` | 完整离线接口手册，支持搜索与浏览器打印 |
| `guide.zh-CN.html` / `GUIDE.zh-CN.md` | 中文接入指南 |
| `guide.ja.html` / `GUIDE.ja.md` | 日本語接入指南 |
| `guide.en.html` / `GUIDE.en.md` | English 接入指南 |
| `guide.de.html` / `GUIDE.de.md` | Deutsch 接入指南 |
| `guide.fr.html` / `GUIDE.fr.md` | Français 接入指南 |
| `openapi.json` | 默认英文 OpenAPI 3.1 接口描述；API 工具仍需自行实现 HMAC 签名 |
| `openapi.zh-CN.json` | 中文 OpenAPI；路径、字段、枚举及签名要求与英文相同 |
| `examples/local_api.py` | 配对与聊天示例；文字模式使用 Python 3 标准库，麦克风模式依赖见脚本说明 |
| `examples/external_tools.py` | 外部工具注册、连接及调用结果回传示例 |
| `examples/media_session.py` | 临时图片、音频和视频会话示例 |
| `manifest.json` | 设备版本、源码提交、生成时间及文件 SHA-256，用于确认文档批次 |

OpenAPI 的 `info.version=1` 是现有接口描述版本；设备软件版本为 1.6.0，另记在 `x-device-version` 和文档包清单中。

五种语言的接入指南均随包提供 HTML 和 Markdown，可从 `index.html` 顶部选择语言。完整接口参考的导航为中文，接口描述默认英文；OpenAPI 提供默认英文 `openapi.json` 和中文 `openapi.zh-CN.json`。接口路径、字段名、枚举值和示例代码保持一致。

## 1. 准备设备

1. 确认设备已安装 1.6.0，并已登录用户账号。
2. 电脑与设备在可互通的局域网中，准备设备 IP。默认 HTTP 地址是 `http://DEVICE_IP:8081`，如设备修改过端口，以实际端口为准。
3. 发送对话前，先唤醒并解锁设备。接口使用设备当前用户和当前角色槽位。
4. 先用下面的在线检测确认网络连通，再运行示例完成配对。

```sh
curl http://DEVICE_IP:8081/v1/api/auth/ping
```

HTTP 基础路径为 `/v1/api`。完整请求地址示例：`http://DEVICE_IP:8081/v1/api/data/user/system`。示例脚本的设备地址只填主机及端口，不加 `/v1/api`。

## 2. 先跑通文字对话

解压后在文档包目录中执行，Python 示例：

```sh
python examples/local_api.py http://DEVICE_IP:8081 "你好" --credentials credentials.json
```

首次运行会让设备显示配对码，在终端输入该码。示例会保存凭据，读取当前槽位，提交一轮文字对话，再查询并输出角色回复。后续运行复用同一个凭据文件。

需要设备播放回复时，增加 `--speech device`：

```sh
python examples/local_api.py http://DEVICE_IP:8081 "你好" --credentials credentials.json --speech device --timeout 180
```

默认 `--speech none`，不播放回复语音；默认等待 120 秒。`device` 模式的 `completed` 表示生成与设备播放均已完成。超时不等于服务端轮次已取消，示例会输出轮次编号，供继续查询。

示例默认申请 `session` 凭据：有效期 12 小时，设备服务重启后失效。需要现有长期凭据时，在首次配对增加 `--auth-scope permanent`。session 可调用公开用户接口；管理接口要求 permanent 凭据，设备功能权限仍然适用。旧凭据文件不会被覆盖或自动升级；重新配对时请指定新的文件名。凭据包含设备访问密钥，应由使用者自行保管，不随问题反馈或文档包转发。

## 3. 自行实现配对

以下五个操作无需 HMAC，其余公开 HTTP 操作需要签名：

| 方法 | 路径（含基础路径） | 用途 |
| --- | --- | --- |
| GET | `/v1/api/auth/ping` | 在线检测 |
| GET | `/v1/api/auth/bootstrap` | 获取当前 `user_id`、登录状态及语言 |
| POST | `/v1/api/auth/pair/request` | 请求在设备上显示配对码，返回 `challenge_id`，请求有效期 5 分钟 |
| POST | `/v1/api/auth/pair/confirm` | 提交配对码，获取凭据 |
| POST | `/v1/api/auth/pair/cancel` | 取消配对请求 |

确认配对的 JSON 正文：

```json
{
  "challenge_id": "使用 pair/request 返回的值",
  "pairing_code": "设备显示的配对码",
  "auth_scope": "session"
}
```

`auth_scope` 支持 `session`、`permanent`；接口省略时默认 `permanent`，示例显式使用 `session`。保存响应 `data.local_secret`、`data.local_session_id`、`data.local_session_expires_at_ms`、`data.auth_scope`，以及 bootstrap 的 `data.user_id`。脚本和原生应用可以不发送 `Origin/Referer`；浏览器接入仍受设备已有来源检查约束。

## 4. 每个业务请求都要签名

规范串按以下顺序拼接，分隔符为 `|`，末尾没有换行：

```text
METHOD|PATH|TIMESTAMP|NONCE|PAYLOAD_MODE|BODY_HASH|USER_ID
```

```text
signature = hex_lower(HMAC_SHA256(UTF8(local_secret), UTF8(canonical)))
```

| 请求头 | 填写方式 |
| --- | --- |
| `X-Local-Sig` | 上述签名，64 位小写十六进制字符串 |
| `X-Local-Timestamp` | Unix 秒数字符串，允许与设备时间相差 300 秒 |
| `X-Local-Nonce` | 每次请求生成新随机值，推荐 UUID；长度 1～128，不能含竖线或换行 |
| `X-Local-Payload-Mode` | 按下表选择 |
| `X-Local-Body-Hash` | 按下表填写 |
| `X-Local-User-Id` | bootstrap 返回的当前 `user_id` |
| `X-Local-Session-Id` | 配对返回的 `local_session_id` 非空时必传 |

`METHOD` 使用大写；`PATH` 使用含 `/v1/api` 的 URL 解码后路径，不含主机和查询字符串。密钥使用 `local_secret` 原始字符串的 UTF-8 字节，不能先做十六进制解码。签名和发送必须使用完全相同的正文字节，不能签名后再次格式化 JSON。

| 正文类型 | PAYLOAD_MODE | BODY_HASH |
| --- | --- | --- |
| 无正文 | `empty` | 空字节的 SHA-256 小写十六进制值 |
| JSON | `json` | 实际 JSON 正文字节的 SHA-256 小写十六进制值 |
| 文件表单 | `multipart` | 固定字符串 `UNSIGNED-PAYLOAD` |
| zstd 压缩上传分块 | `zstd-stream` | 实际压缩分块字节的 SHA-256 小写十六进制值 |
| gzip 压缩上传分块 | `gzip-stream` | 实际压缩分块字节的 SHA-256 小写十六进制值 |

无正文哈希：`e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`。压缩分块使用 `application/octet-stream`，仅用于手册明确支持的上传接口。文件表单应由 HTTP 库生成包含 boundary 的 `Content-Type`。

## 5. 统一对话 API

| 方法 | 路径（相对于 `/v1/api`） | 用途 |
| --- | --- | --- |
| POST | `/data/user/chat/turns` | 创建一轮对话 |
| GET | `/data/user/chat/turns/{turn_id}` | 查询本轮状态、转写和回复 |
| POST | `/data/user/chat/turns/{turn_id}/input/complete` | 结束设备麦克风输入，继续生成回复；空正文 |
| POST | `/data/user/chat/turns/{turn_id}/cancel` | 取消本轮输入、生成及播放；空正文 |

先调用 `GET /data/user/slot` 取得 `data.slot_number`。创建文字对话的正文如下，其中 `request_id` 每个新业务轮次使用新的 UUID，`slot` 换成当前槽位：

```json
{
  "request_id": "3f743580-bd45-4c24-96c0-61a67ac5f741",
  "slot": 1,
  "input": {"type": "text", "text": "你好"},
  "output": {"speech": "none"}
}
```

| 输入 | Content-Type | 内容 |
| --- | --- | --- |
| `text` | `application/json` | `input.text` 必填，去除首尾空白后非空，最多 16000 字符 |
| `device_microphone` | `application/json` | 由设备麦克风录音，不传 `text`；结束输入使用 `input/complete` |
| `image_text` | `multipart/form-data` | `request` 字段为上述配置的 JSON 字符串，`input.type=image_text`，可附 `text`；文件字段为 `image` |
| `audio_file` | `multipart/form-data` | `request` 字段为配置 JSON 字符串，`input.type=audio_file`，不传 `text`；文件字段为 `audio` |

图片或 WAV 文件最多 1 MiB。WAV 仅支持 PCM16、16 kHz、单声道、最长 30 秒。`output.speech` 必填，可为 `none`、`device` 或 `client`；请求不接受未知字段。

可选 `interrupt` 默认为 `true`：轮次开始时设备正在对话、播报或执行任务，会先结束当前会话。设为 `false` 时不打断，轮次以 `failed` 结束，`error.code=DEVICE_BUSY`（`retryable=true`），本轮已写入的用户消息从历史中撤回；稍后重试即可。

可选 `retention` 决定本轮在设备上留下什么，默认为 `full`：写入聊天记录，并更新对话状态、摘要、长期记忆和日记（它们都从本轮聊天记录抽取）。`chat` 只写聊天记录，不做这些更新。`none` 什么都不留：不写聊天记录、不保存图片、不做上述更新，回复只出现在本轮的 `assistant_text` 中，`user_chat_ids`、`assistant_chat_ids` 为空。三种取值下，生成回复时都照常读取已有的聊天记录和记忆。

可选 `tools` 决定本轮模型能否调用工具，默认为 `all`：模型可以调用本次会话可用的设备工具和外部工具。设为 `none` 时本轮不向模型提供任何工具，适合只需要一句回复、不应触发设备动作或外部任务的轮次。

还支持 `input.type=audio_stream`：使用 JSON，声明 `audio.format=pcm_s16le`、`sample_rate=16000`、`channels=1`、`container=raw`，音频最长 30 秒。音频流输入或 `speech=client` 须在受理后 15 秒内，以签名 GET 连接 `/v1/ws/chat/turns/{turn_id}/audio` WebSocket 并等待 `ready`。客户端输出按元数据接收 MP3、WAV 或 Ogg Opus，在 `audio.end` 后发送最终序号的 `audio.ack`；这表示收齐音频，不表示播放结束。断开即结束任务，不支持重连或重放。完整消息格式见 OpenAPI。

```sh
python -m pip install websocket-client sounddevice av
python examples/local_api.py http://DEVICE_IP:8081 --mic --seconds 10 --credentials credentials.json --timeout 180
```

创建成功从响应 `data.turn_id` 取编号，建议每秒查询一次对应轮次。终态为 `completed`、`failed`、`cancelled`；`completed` 时读取 `data.assistant_text`，失败时读取 `data.error`。`transcript` 提供转写，`user_chat_ids`、`assistant_chat_ids` 关联历史记录。中间状态不应当作最终成功。

状态还包括 `queued`（等待执行）、`starting`（建立会话）、`receiving_input`（接收录音或音频流）、`thinking`（等待生成）、`responding`（生成或交付回复）、`cancelling`（取消处理中）。`input_complete` 仅表示输入结束，`transcript` 可早于终态出现。`device_microphone` 最长录音 30 秒后自动结束输入，`input/complete` 用于提前结束。`speech=none` 仍会调用模型，只是不朗读回复；是否保存对话由 `retention` 决定。每次请求前可读取 `/data/user/device/state` 检查当前准入条件。

`202` 仅表示受理。同一凭据、同一 `request_id`、相同内容在暂存期限内重放返回原轮次；内容不同返回 `409 REQUEST_ID_CONFLICT`。重试 HTTP 请求仍须生成新的签名 nonce。`interrupt=true` 时，新轮次会停止旧会话；相同请求重放不会重复打断。

轮次终态保留 30 分钟，最多暂存 256 个；设备服务重启会丢失轮次和幂等记录。角色或用户切换、凭据失效、清空历史会使旧轮次停止或不可访问。取消只能作用于该凭据创建的指定轮次。

### 独立文本播报（TTS）

将外部应用已经确定的文本直接交给当前角色朗读，例如 Home Assistant 的执行结果。这个接口不调用 LLM，不生成角色回复，也不写入聊天历史。

提交 JSON 到 `POST /data/user/tts/utterances`；所有路径相对于 `/v1/api`，沿用上文的 HMAC 签名：

```json
{
  "request_id": "e14eec98-5f67-445f-b605-bcc0b23f8f75",
  "slot": 1,
  "text": "客厅灯已关闭。"
}
```

通过 `GET /data/user/tts/utterances/{utterance_id}` 查询。取消用 `POST /data/user/tts/utterances/{utterance_id}/cancel`，正文为空。

202 仅表示受理，读取 `data.utterance_id`。状态为 `queued`、`synthesizing`、`playing`、`cancelling`，终态为 `completed`、`failed`、`cancelled`；只有合成和设备播放都结束才是 `completed`，失败详情见 `data.error`。`text` 去除首尾空白后须为 1–16000 字符，JSON 正文最多 128 KiB；使用当前槽位和当前角色的声音、语言及音量。设备须唤醒且解锁；新播报会打断当前前台会话，新对话也可打断播报，不设播放队列。

同一凭据、request_id 和相同内容重试返回原任务（200），不同内容返回 409；request_id 与聊天共享命名空间。终态保留 30 分钟，与聊天共用 256 条暂存；服务重启不保留。凭据失效、用户/角色切换或清除历史会终止或使旧任务不可访问，任务最长 180 秒。接口不返回音频文件或音频流。

```sh
python examples/local_api.py http://DEVICE_IP:8081 "客厅灯已关闭。" --credentials credentials.json --tts --timeout 180
```

`playing` 以当前播放流的设备开播回执为准。允许动画时，随机匹配一个 Talking 动画。取消请求受理返回 202；已在取消中或已取消返回 200，已完成或失败返回 409。客户端等待超时不会自动取消设备任务。

## 6. 其他业务与结果确认

完整路径、字段和响应以离线手册和 OpenAPI 为准：

| 分类 | 内容 |
| --- | --- |
| 接入与配对 | 在线检测、配对及取消 |
| 设备与状态 | 系统信息、能力、设备与对话状态、AI 语言 |
| 角色与主动交互 | 角色属性、槽位、模型服装上传与导出、加载、重置及主动交互 |
| 对话与历史 | 统一对话、历史查询删除、聊天图片 |
| 记忆与日记 | 查询与编辑 |
| 闹钟与备忘录 | 资源管理及闹钟启停 |
| 音乐 | 上传、文件管理、播放及状态 |
| 声音 | 能力、独立文本播报、固定文案试听、控制及声音克隆 |
| 灯光 | 作品、预览、编译、播放、应用及总开关 |
| 模型服务 | 服务商目录、账号与测试、能力路由、系统提示词、架构及后端切换、最近运行错误 |

配对不会新增设备功能权限。先读取 `/data/user/system` 的 `capabilities`，再调用相应功能；手册逐项标出 `x-capabilities`。实验室 API 文档页面及 AI 服务配置向所有功能标签开放，接口继续使用现有配对签名鉴权。

接口成功可能表示保存或命令入队，不能统一解释为设备动作已完成。对话查询轮次；模型上传、应用、导出等查询相应任务或加载状态；音乐查询播放状态；声音试听查询 `/data/user/tts_audition/state`。声音试听仅接受 `voice_id`、可选 `group_key`，使用设备当前 AI 语言的固定文案，不接收自定义播报文字。

模型文件最多 200 MiB、声音克隆文件最多 10 MiB；音乐批量请求最多 200 MiB、64 个文件。普通签名正文及压缩上传分块最多 8 MiB。表单总请求还需包含字段开销，各接口的完整约束以手册为准。

可选实时状态通道：`/v1/ws/events`。使用签名 GET 握手，查询参数为 `sig`、`timestamp`、`nonce`、`payload_mode=empty`、`body_hash`、`user_id`，以及非空时的 `session_id`。签名路径为 `/v1/ws/events`，不含查询字符串。通道推送状态快照，不提供聊天生成文字流；凭据失效或用户切换等情况会关闭连接。

建连收到 `ready` 后，发送 `{"type":"subscribe","request_id":"sub-1","data":{"events":["chat_history","chat_turn"]}}`。等待 `result` 成功后接收事件；`events` 是完整替换集合，空数组取消全部订阅。重连后重新订阅，再通过 HTTP 恢复状态。公开契约仅包含 Release 构建和 RELEASE 标签均开放的能力；原始 OpenAPI 描述默认英文。

## 7. 常见问题与反馈

| 现象或返回码 | 处理 |
| --- | --- |
| 无法连接 | 检查 IP、端口和局域网互通，先调用 ping |
| `401 REQUEST_EXPIRED` | 对齐电脑和设备时间，重新生成时间戳及签名 |
| `401 NONCE_REUSED` | 每个 HTTP 请求都生成新 nonce，包括查询和重试 |
| `401 LOCAL_SESSION_EXPIRED` | 使用新的凭据文件重新配对 |
| `401 LOCAL_USER_MISMATCH` | 核对当前用户，重新获取 bootstrap 并配对 |
| `401 LOCAL_SECRET_MISMATCH` / `INVALID_SIGNATURE` | 核对原始密钥、签名路径、正文字节及各请求头 |
| `403` | 查看响应错误字段及当前设备功能权限；配对不能绕过权限 |
| `409 SLOT_MISMATCH` | 重新读取当前槽位 |
| `409 DEVICE_BLOCKED` / `AMBIENT_MODE_BLOCKED` | 查看返回 `reason`，确认设备已唤醒、解锁且当前模式允许对话 |
| `413` / `415` | 检查大小、Content-Type、输入输出类型和 WAV 格式 |
| `503` | 按错误码确认服务是否就绪或暂存轮次数是否达到限制 |

OpenAPI 的错误响应 `code.enum` 列出聊天、TTS、闹钟、备忘录及设备克隆录音的本地 HTTP 错误码；操作的 `x-error-codes.http` 提供机器清单，`x-error-codes.task` 列出已知异步错误，提供方可能扩展异步错误码。HTTP 202 后的失败应读取 `data.error`。配对来源不允许返回 `403 PAIRING_SOURCE_NOT_ALLOWED`；幂等内容冲突返回 `409 REQUEST_ID_CONFLICT`；不存在的备忘录返回 `404 MEMO_NOT_FOUND`。

`x-error-codes` 还涵盖 AI Provider、模型任务、角色重置、记忆及音乐的已知错误；日记的旧格式错误标识列在 `legacy_error`，实际位于响应 `error` 字段。已知异步错误含 `NO_SPEECH`；`DEVICE_INTERRUPTED.reason=vad_start` 表示检测到语音开始，reason 保持可扩展。

不同业务的错误结构存在差异，不保证全部使用相同字段。先判断 HTTP 状态，再读取该接口的 `code`、`message`、`reason` 等；不要盲目重试写操作。

反馈时提供：设备软件版本、文档包 `manifest.json` 中的源码提交、发生时间及设备时区、HTTP 方法和路径、状态码、脱敏后的响应、相关 `request_id` / `turn_id` 和设备现象。不要发送 `local_secret`、凭据文件或签名请求头。

管理接口、调试音频、诊断日志和内部维护接口不属于本次对外契约。本文档包核对的是源码接口契约；工程编译、真实设备调用及云端完整链路需分别验证。
