# docs/01-intake.md

## 第一段 · 多模态接入：语音 / 文本 / 照片，一个入口全接住

### 结论先行

本段解决"客户到底发了什么"：把 Telegram 消息按类型**三岔分流**——语音去转写、文本直读、其余一律礼貌拒绝。语音链路里藏着两个工程细节：**二进制文件要修文件名才能过 Whisper**、**转写结果要带上"这是谁发的"上下文**。

```
Telegram 任意消息
        │
        ▼
   ┌─ Has Voice? ──是──▶ Get Voice File（拿音频二进制）
   │                          │
   │                          ▼
   │                   Fix Audio Filename（Code：归一 data 键 + 补扩展名）
   │                          │
   │                          ▼
   │                   Transcribe (Whisper)（Groq /audio/transcriptions）
   │                          │
   │                          ▼
   │                   Extract Transcript (Voice)（Code：转写文本 + 身份）
   │
   ├─ Has Text? ──是──▶ Extract Transcript (Text)（Code：直读文本 + 身份）
   │
   └─ 都不是 ───────▶ Unsupported Message Reply（照片/贴纸/视频 → 拒绝文案）
```

---

### 1. 触发器：一条消息进来，先问三个问题

`telegramTrigger`（Voice Message Trigger）在有人给 bot 发消息时触发。紧接着的 `Has Voice?` 判断 `message.voice` 是否存在，`Has Text?` 判断 `message.text` 是否存在。两个 if 叠起来就是三岔：

| 消息类型 | 走向 | 为什么 |
|---|---|---|
| 语音 `voice` | 转写链 | 客服场景语音占比高（走路、开车、着急说不清） |
| 文本 `text` | 直读链 | 大多数消息就是文字 |
| 照片 / 视频 / 贴纸 / 位置… | Unsupported 回复 | **宁可不答，不要瞎答**——客服 bot 对看不懂的内容给个台阶最安全 |

> 🧠 设计思想：**"拒绝"也是一种回复策略**。模板没有尝试让 LLM 看图，而是明确告诉用户"目前只支持语音和文字，请打字描述"，把期望管住。

### 2. 语音链：下载 → 修文件名 → 转写

**Get Voice File**：n8n Telegram 节点把语音下载为二进制（`binary` 数据）。注意：Telegram 的 voice 消息本体是个 `file_id`，需要先调 `getFile` 换真实文件——n8n 节点把这步封装了，你拿到手的就是二进制。

**Fix Audio Filename（Code）**——名字不起眼，内容全是坑：

```js
// 要点一：把 binary 数据归一到 data 键（防御不同来源的字段名差异）
// 要点二：修文件名——Whisper 上传要求可识别的扩展名/MIME
// 要点三：没有 binary 就主动抛错（让错误网接管，而不是带病往下传）
```

这个节点有 3 条核心防御逻辑（对应我们单测里的 5 个断言）：
1. `data` 键直传 + 改名改类型；**非 `data` 键被归一到 `data` 且旧键删除**（防止 Telegram / 其他来源字段名漂移）；
2. 多个键并存时**优先 `data`**；
3. 无 binary / binary 为空对象 → **主动抛错**——把问题丢给错误网，而不是给 Whisper 传一个空文件然后等一个莫名其妙的 400。

**Transcribe (Whisper)**：HTTP Request 调 Groq `POST /audio/transcriptions`，multipart 上传修复后的文件，模型 `whisper-large-v3-turbo`。注意**这是整条流里唯一一个"文件上传"型接口**，跟后面所有 JSON 对话接口长得都不一样——调试时最容易在这里栽跟头（返回的不是 JSON 数组而是一个字符串 `{text: "..."}`）。

### 3. 文本链与归一：让两条路汇成同一个 transcript

**Extract Transcript (Text)** 把 `message.text` 提出来；**Extract Transcript (Voice)** 把 Whisper 返回的 `text` 提出来。两个节点输出**同一套字段**：

```
transcript   ← 用户说了什么（语音=转写结果 / 文本=原文）
chatId       ← 用户是谁（后续所有回复、锁、留痕都靠它）
first_name   ← 昵称（缺失时回退 "Customer"，LLM 上下文用）
```

> 🧠 设计思想：**下游只认一套字段**。语音和文本两条路物理不同，但汇合点之后（分类、路由、留痕）完全不知道也不需要知道消息来自哪条路。这就是"接口归一"——如果每个分支各带各的字段名，后面每个节点都要写 if 判断，复杂度爆炸。

### 4. chatId 从哪来？——它得"活"到最后

分类、回复、写锁、留痕……后面每一段都需要知道"这条消息是谁发的"。模板的策略是：**每次组装上下文时都把 chatId 带上下传**，一直到 `Parse Classification JSON` 的输出里还保留着它。万一中途某节点出错，错误网还有**四级回退链**兜底（docs/05 详述）：

```
① Parse Classification JSON 的 json.chatId
② Extract Transcript (Voice) 的 json.chatId
③ Extract Transcript (Text) 的 json.chatId
④ Voice Message Trigger 的原始 chat.id
```

全部缺失 → `unknown` + 默认 transcript。**用户 ID 是客服系统的生命线，宁可多存三份，不可丢一次。**

---

## 坑点提示

- **文件名 / MIME 是语音链最大暗礁**：Telegram 给的音频常无扩展名，直接传 Whisper 会失败或识别成未知格式。修文件名节点不是洁癖，是刚需。
- **`if` 的 false 分支别漏接**：`Has Voice?` 的 false 要接到 `Has Text?`，`Has Text?` 的 false 才接 Unsupported——漏一根线，照片消息就会掉进"无输出"黑洞（n8n 静默吞掉，用户永远等不到回复）。
- **照片消息不硬答**：模板对照片直接拒绝。如果你未来要支持 OCR / 截图问答，是在这里加新分支，而不是改 Unsupported 的文案。

## 动手练习（详见 exercises 第 1 题）

1. 找出 `Fix Audio Filename` 的 Code，给"binary 为空对象"补一个自己的抛错文案，想想错误网会怎么接住它；
2. 给自己的 bot 分别发：语音、纯文本、一张照片，观察三条路径和三类回复；
3. 思考：如果一条消息**同时带文字和语音**（Telegram 支持），模板会优先走哪条？为什么？（提示：看 `Has Voice?` 的位置）