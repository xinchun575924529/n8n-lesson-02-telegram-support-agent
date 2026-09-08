# docs/04-routing.md

## 第四段 · 路由：按意图给消息分配"不同身价"的模型

### 结论先行

升级段拦下了"要人管"的消息，剩下的常规问题按意图路由到**不同档位的模型**：FAQ 用 8B 快模型（max_tokens 200，毫秒级回复），交易 / KYC / 安全 / 交易类用 70B 大模型（答得深、给步骤）。**同一个 Groq key、同一个接口，只换 model 名和 prompt 人设——分级客服的算力成本就是这么省的。**

```
Parse Classification JSON（intent 判决）
        │
        ▼
Route by Intent（switch，5 分支）
   ├── faq ──────────────▶ Fast Model Reply (FAQ)        llama-3.1-8b-instant（快/便宜）
   ├── transaction_issue ─▶ Contextual Reply (Txn/KYC)   llama-3.3-70b-versatile（强）
   ├── account_kyc ──────▶ Contextual Reply (Txn/KYC)    llama-3.3-70b-versatile（强）
   ├── wallet_security ──▶ Strong Model Reply (Sec/Trade) llama-3.3-70b-versatile（强）
   └── trading_swap ─────▶ Strong Model Reply (Sec/Trade) llama-3.3-70b-versatile（强）
        │
        ▼
Extract Reply Text（从 OpenAI 兼容响应提纯 content）
        │
        ▼
（Send Reply to User → 交付段留痕）
```

---

### 1. switch 分流：intent 是唯一的钥匙

`Route by Intent` 按分类段的 `intent` 字段分 5 路。注意模板的归类策略：**5 个 intent 只对应 2 个模型档位**（再加分类用的 8B 共 3 个 LLM 角色）：

| intent | 走哪路 | 模型档位 | 理由 |
|---|---|---|---|
| `faq` | Fast Model Reply | 8B + 200 token 上限 | 高频、简单、答案可短 |
| `transaction_issue` | Contextual Reply | 70B | 涉及具体交易上下文，要仔细 |
| `account_kyc` | Contextual Reply | 70B | 身份认证规则多、答错麻烦 |
| `wallet_security` | Strong Model Reply | 70B | 安全类必须给足步骤和警告 |
| `trading_swap` | Strong Model Reply | 70B | 交易操作，答错就是钱 |

> 🧠 成本工程的本质：**不是所有问题都值得用最强模型**。客服流量里 FAQ 通常占 60%+——把它们全部打在 8B 上，成本可能省一个数量级，而用户根本感知不到差别（FAQ 本来就不需要深度推理）。70B 只留给"答错了会出事"的问题。

### 2. 三个 LLM 节点的同与不同

三个 HTTP 节点**长得一模一样**（同一 URL `/chat/completions`、同一凭证），区别只有两处：

| 节点 | model | prompt 人设 |
|---|---|---|
| Fast Model Reply (FAQ) | `llama-3.1-8b-instant` | "friendly, concise support assistant for CBox"，教用户创建钱包等基础操作，max_tokens 200 限制长度 |
| Contextual Reply (Transaction/KYC) | `llama-3.3-70b-versatile` | 带上用户的具体交易 / 账户上下文，解释发生了什么、怎么解决 |
| Strong Model Reply (Security/Trading) | `llama-3.3-70b-versatile` | 强调安全：警告不要分享私钥、给出官方安全步骤、必要时引导升级人工 |

prompt 都带上了上游上下文（transcript + chatId + first_name），让模型能"接着聊"而不是瞎答。

> 🔧 **换底技巧（重要）**：这三个节点是 OpenAI 兼容接口。想换 DeepSeek / OpenAI / 本地 Ollama，只需要改 `url` 的域名 + `model` 名 + 凭证，**prompt 和消息结构一字不用动**。Groq 免费额度用完时，这是最省事的降级路线。

### 3. Extract Reply Text：把模型的"套娃响应"剥开

Groq 返回的是标准 OpenAI 格式：`choices[0].message.content` 才是正文。`Extract Reply Text` 做三件事：
1. 从响应里安全取出 content（结构不对时兜底空字符串）；
2. 把上游的 `chatId` / `transcript` 等上下文**原样透传**（跟 Parse Classification JSON 一个套路——每次加工都别丢上下文）；
3. 输出干净的 `replyText` 给发送节点。

### 4. ⚠️ 延伸坑：switch 没命中 = 又一个"0 项输出"黑洞

还记得 docs/03 的头号大坑吗？**上游 0 项输出 → 下游静默不跑**。switch 同理：如果分类模型吐出一个 schema 之外的 intent（比如你加了 `refund` 却忘了改 switch），switch 没有对应分支 → **0 项输出 → 后面的发送节点静默不跑**，用户消息又石沉大海。

```
switch 未命中任何分支 ──▶ 输出 0 项 ──▶ Send Reply to User 不执行（静默！）
```

所以"加一个新 intent"永远是一套组合拳：分类 prompt 的 schema → Parse 的回退表 → switch 分支 → 路由目标模型 → （可选）日志字段。漏一环就是一次静默吞消息。**这也是为什么 Parse 的垃圾回退要落在 `faq`**——faq 是 switch 里必然存在的分支，兜底值必须指向一个"永远有出口"的路由。

---

## 坑点提示

- **5 个 intent 走 2 档模型是刻意设计**，不是作者偷懒：分类用 8B 已经做过一轮"轻重分离"，路由这层只做"出口选择"，两层各司其职。
- **max_tokens 是成本闸门**：FAQ 节点锁 200 token 不只是防废话，更是防模型"话痨"烧钱——长回复在客服场景几乎没价值。
- **改模型名后务必重跑一次端到端**：换 baseURL / model 是最常见的"看起来改了、其实没生效"操作（凭证引用错、节点没保存、import 后未 publish 都会导致静默用旧配置）。

## 动手练习（详见 exercises 第 4 题）

1. 把 Fast Model Reply 的 model 临时改成不存在的名字，发一条 FAQ，观察错误网如何接住（对照 docs/05 的错误链）；
2. 新增 `refund` intent：列出你要动的全部节点，并设计 switch 该把它接到哪档模型；
3. 对比三个 LLM 节点的 system prompt，总结"上下文型"和"安全型"人设的差异，思考为什么安全型要主动给"不要分享私钥"这类警告。