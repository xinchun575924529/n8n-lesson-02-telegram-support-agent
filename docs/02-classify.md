# docs/02-classify.md

## 第二段 · 理解：让 LLM 先"表态"，再决定怎么办

### 结论先行

分类段的产出是一个**结构化判决**：这条消息的 `intent`（要办什么事）、`sentiment`（用户什么情绪）、`security_risk`（有没有安全风险）、`confidence`（模型多自信）。后续升级还是路由，全靠这份判决。本段最值钱的部分不是"调了一次 LLM"，而是 **`Parse Classification JSON` 的防御式解析**——它把 LLM 的输出当"不可信输入"处理，保证任何脏输出都不会炸掉整条客服链。

```
Extract Transcript（上游归一字段）
        │  transcript + chatId + first_name
        ▼
Classify Intent & Sentiment（HTTP → Groq chat/completions）
        │  llama-3.1-8b-instant · temperature=0 · response_format=json_object
        │  system prompt 硬性规定 JSON schema + security_risk 触发词
        ▼
Parse Classification JSON（Code：四件套防御）
        │  输出：{ intent, sentiment, security_risk, confidence, chatId, ...prior 透传 }
        ▼
（去 Escalation 段查锁 / 判定升级）
```

---

### 1. 调用端：三个参数把"自由发挥"关进笼子

`Classify Intent & Sentiment` 是个 HTTP Request 节点，三个关键设置：

| 参数 | 值 | 作用 |
|---|---|---|
| `model` | `llama-3.1-8b-instant` | 分类用 8B 就够——快、便宜，情绪识别不需要 70B |
| `temperature` | `0` | **分类任务不许有随机性**，同一句话每次判定必须一致 |
| `response_format` | `{ "type": "json_object" }` | 强制模型输出 JSON（Groq 的 OpenAI 兼容模式支持） |

**system prompt 是真正的裁判规则**，它规定输出**必须是这个形状**：

```
{"intent": "faq|transaction_issue|account_kyc|wallet_security|trading_swap",
 "sentiment": "neutral|frustrated|angry",
 "security_risk": true|false,
 "confidence": 0.0}
```

并给 `security_risk` 列出了**具体触发词**：未授权交易、异常扣款、私钥泄露、被盗……（"我密码好像泄露了"哪怕语气平静，也必须判 risk=true——安全风险看**内容**不看**情绪**，这是后续升级逻辑的前提）。

### 2. 解析端：把 LLM 当"不可信输入"

模型嘴上答应输出 JSON，实际可能给你：

- ✅ 干净的 JSON：`{"intent":"faq","sentiment":"neutral","security_risk":false,"confidence":0.9}`
- 🐛 包了 markdown 围栏：```` ```json {…} ``` ````
- 🐛 布尔值写成字符串：`"security_risk": "true"`
- 🐛 前面夹废话："Here is your JSON: {…}"
- 🐛 完全不是 JSON（超长、截断、模型抽风）

`Parse Classification JSON` 用四件套把以上全部收拾干净：

| 防御 | 做法 | 对应单测 |
|---|---|---|
| ① 提取 | 从响应里抽出第一段 `{...}`，忽略前后废话 | 裸 JSON 正确解析 |
| ② 清洗 | 剥掉 ` ```json ` 围栏再 parse | 围栏被清洗 |
| ③ 回退 | 解析失败 → **静默回退** `faq / neutral / security_risk=false / confidence=0`，绝不抛错 | 垃圾文本回退通过 |
| ④ 归一 | `security_risk` 用 `!!` 双非转成真布尔（`"true"`→`true`、`1`→`true`、`"false"`→`false`） | 真值归一为布尔 |

> 🧠 为什么回退值是 `faq/neutral` 而不是抛错？——客服场景"宁可答错方向，不能没有回复"。faq 是最安全、最不可能升级的默认档：一条无法解析的消息按 FAQ 答一句通用帮助，比把用户晾着或误升级人工强得多。

最后，节点把**原始上下文原样透传**（`prior` 展开保留）——chatId、transcript 继续跟着走，因为后面的升级、路由、留痕全都要用。**每个解析节点都要记住：你只是"加了一层判断"，不是"把消息拦下来"**。

### 3. 为什么分类错位会引发连锁灾难？

分类是整条流水线的"红绿灯"：
- 该升级没升级（把 `security_risk=true` 判成 false）→ 用户被盗刷没人管 → **安全事故**；
- 不该升级却升级（把 faq 判成 angry）→ 人工被无关消息轰炸 → 30 分钟锁把真问题也挡在外面；
- 解析直接抛错 → 走错误网 → 用户收到兜底回复（本课 E1 场景演示的就是这条，docs/05 有完整链路）。

所以分类段在 L2 验证时配了 **5 个解析断言** + 1 个端到端"分类失败"场景（E1）——验证的不是"模型答得对不对"，而是"模型答得再烂，流水线都不死"。

---

## 坑点提示

- **`json_object` 模式 ≠ 一定合法 JSON**：它只是提高概率。围栏、截断、字符串布尔依然会出现——解析器必须全兜住。
- **回退值本身要有业务含义**：`faq/neutral/false` 是精心选的"最安全默认"，不是随便填的。设计任何"解析失败回退"前先问：回退到哪一档对业务伤害最小？
- **temperature=0 只对分类类任务对**：创意文案、话术生成如果也锁 0，输出会机械重复。别把这条经验搬到所有 LLM 节点。

## 动手练习（详见 exercises 第 2 题）

1. 手工把分类 HTTP 节点的响应改成脏数据（围栏 / 字符串 true / 纯废话），观察 Parse 节点的三种出路；
2. 给 system prompt 新增一个 intent（比如 `refund`），列出你需要在哪些节点同步修改（提示：不止分类 prompt 一处）；
3. 想想：`confidence` 字段目前没被任何下游使用——如果要用它做"低置信度转人工"，应该插在哪两个节点之间？