# docs/05-delivery-errors.md

## 第五段 · 交付与错误网：回复发出去、全程留痕、出错还有人管

### 结论先行

正常消息走"发送 → 组装日志 → 写表"三连；任何转写 / 分类 / 回复生成节点出错，走**错误网三连**：Slack 告警（给运维）→ 错误表留痕（给审计）→ 兜底回复（给用户）。本段是整课"含金量"最高的地方——模板在这里藏了**两个我们实测实锤的 bug**（日志列映射错位、错误网盲区），看懂它们，你就掌握了"验证模板"的方法论。

```
【正常路径】
Send Reply to User（客服 bot 回复文本）
        │
        ▼
Prepare Log Data（Code：组装日志字段）
        │  escalated / replyText / securityRisk(改名) / timestamp
        ▼
Log Interaction（写 Interactions 表）

【升级路径（模板原版）】
Send Voice Reply to User ──▶ Log Interaction  ⚠️ 跳过 Prepare Log Data！（Bug B）

【错误路径】
（Transcribe / Classify / 三个 Reply / TTS 任一报错）
        │
        ▼
Handle API Error（Code：chatId 四级回退链 + 组装错误 payload）
   ├──▶ Alert Slack (Error)      （告警运维，含失败节点 + 错误详情）
   ├──▶ Log Error to Sheet       （错误表留痕）
   └──▶ Fallback Reply to User   （友好兜底文案）
```

---

### 1. 正常路径：Prepare Log Data 是"日志统一出口"

`Prepare Log Data` 把散落各处的字段组装成一条**规整的日志行**：
- `escalated`：本次是否升级（普通路径 = false）；
- `replyText`：实际回复了用户什么；
- `securityRisk`：字段改名（分类里的 `security_risk` → 日志里的 `securityRisk`，蛇形转驼峰是跨系统惯例）；
- `timestamp` + `chatId` + `transcript`：时间、用户、内容三要素。

> 🧠 设计思想：**日志要"一个出口、统一 schema"**。如果每个分支各写各的，审计时字段对不上、报表拆不开。prepare → log 两段式，让组装逻辑可单测、可复用。

### 2. ⚠️ Bug A（官方模板实证）：Log Interaction 列映射错位

我们跑 mock 场景 S3（文本·交易问题）时，断言"日志表 intent 列存的是 intent"——**失败了**。查 Interactions 表发现：

| 列名 | 实际存进去的值 |
|---|---|
| `intent` 列 | 存的是 **sentiment**（情绪值） |
| `userName` 列 | 存的是 **transcript**（消息原文） |

**列映射写反/错位了**：节点的字段表达式把 sentiment 填进了 intent 列、把 transcript 填进了 userName 列。后果：**日志表里的 intent 列永远查不到真实意图**，任何基于它的报表、统计、质检全部失真。

```
你以为：Log Interaction 的 intent 列 ← classification.intent
实际上：intent 列 ← classification.sentiment   ← 错位！
         userName 列 ← transcript              ← 错位！
```

**根因猜测**：模板作者复制粘贴字段映射时没对齐列名，且这类 bug 在"能跑通"的表面下极难发现——消息能回复、表能写入、没人看列内容，就永远没人知道写错了。

### 3. ⚠️ Bug B（官方模板实证）：升级链跳过了 Prepare Log Data

正常路径是 `Send Reply → Prepare Log Data → Log Interaction`；但升级路径（docs/03 的语音安抚链）在模板原版里是 **`Send Voice Reply to User` 直接连 `Log Interaction`**——跳过了组装节点！

后果：升级日志行没有 `escalated=true` 标记、没有规整的 replyText，甚至上游字段都不齐（Telegram 发送结果的字段 ≠ 我们准备的日志字段）。**升级记录形同虚设**——最需要审计的"转人工事件"反而最没记录。

> 🧠 两个 bug 的共性：**能跑 ≠ 对**。消息正常回复、表正常写入，字段语义却全错了。这种 bug 只有"对着字段规格做断言"才能抓出来——这就是本课验证方法论的价值：我们给 Log Interaction 写了"intent 列 == 分类的 intent"这类列级断言，S3 一跑就现形。

### 4. 错误网：谁罩着、谁裸奔

`Handle API Error` 的错误 payload 组装有一个**精华设计——chatId 四级回退链**：

```
① Parse Classification JSON.json.chatId   （最完整，走到分类的都有）
② Extract Transcript (Voice).json.chatId  （语音链半路挂的）
③ Extract Transcript (Text).json.chatId   （文本链半路挂的）
④ Voice Message Trigger 原始 chat.id      （入口处就挂的）
全缺 → unknown + 默认 transcript
```

**为什么要回退？** 错误可能发生在流程任何位置，每个位置的"当前上下文"不同：分类节点挂了，它上游的转写结果可能还在；入口就挂了，只有触发器原始消息可用。**兜底回复必须发给正确的用户**——逐级往上找最近的 chatId，是错误处理里"尽量别丢用户"的典范。

错误网三连的职责分离：

| 动作 | 给谁 | 内容 |
|---|---|---|
| Alert Slack (Error) | 运维 | 失败节点名 + 错误信息（E1 实测断言：failedNode = "Classify Intent & Sentiment"） |
| Log Error to Sheet | 审计 | chatId + 时间 + 错误详情（错误表，与 Interactions 分开） |
| Fallback Reply to User | 用户 | 友好文案："系统开小差了，请稍后再试"——**用户永远不该面对裸错误** |

### 5. ⚠️ 错误网盲区（Bug C）：发送节点裸奔

仔细看错误网的连线：罩住的是**转写 / 分类 / 三个 Reply / TTS**（所有"生成内容"的节点），但**四个 Telegram 发送节点（Send Reply / Holding Reply / Notify Human Agent / Send Voice Reply）不在网内**——它们自己出错（bot 被拉黑、网络抖动、消息超长被拒）时，没有错误分支，直接失败且无人知晓。

```
Send Reply to User ──(发送失败)──▶ ❌ 无错误分支！运维不知、用户不知
```

**"最后一公里"的错误最该兜**：回复没发出去，用户会以为客服死了，比 AI 答错更伤信任。这是本课留给你的改造作业（exercises 第 6 题）——把发送节点也挂进错误网，或至少让发送失败走一次告警。

### 6. 三个 bug 一起看：验证方法论 > 模板本身

| Bug | 类型 | 怎么被发现的 |
|---|---|---|
| A 列映射错位 | 字段语义错误 | 列级断言（intent 列 == intent 值） |
| B 升级链不组装日志 | 链路遗漏 | 场景断言（升级事件必须留痕 escalated=true） |
| C 错误网盲区 | 覆盖不全 | 拓扑审查（发送节点无 error 连线） |

**方法论沉淀**：拿到任何 n8n 模板，先别急着接凭证上线，花 30 分钟做三件事——① 把每个 Code 节点的输入/输出字段画成规格表；② 给字段映射写断言（mock 层就能跑）；③ 审查每条链路的"空输出"和"错误输出"两个边界。本课 32 + 42 条断言就是这么来的。

---

## 坑点提示

- **日志字段要"列级断言"**：只验证"写入成功"远远不够——必须验证"每一列的值来自正确的源字段"。Bug A 就是"写成功了但写错了"。
- **升级事件是最该留痕的事件**：转人工 = 可能要担责。escalated 标记、时间、原文、处理人，一个都不能少（Bug B 提醒你检查升级链有没有自己的组装节点）。
- **错误网要覆盖发送节点**：生成内容失败有兜底，发送失败没有兜底 = 用户彻底失联。检查你的错误网是不是也漏了"最后一公里"。
- **Code 沙箱用 fs 要自声明**：n8n task-runner 安全沙箱默认禁 `fs` 等内置模块——Code 里 `require('fs')` 会报 "Module 'fs' is disallowed"。要么在 Code 顶部显式声明并在实例环境放行 `NODE_FUNCTION_ALLOW_BUILTIN`，要么别在 Code 里碰文件。

## 动手练习（详见 exercises 第 5、6 题）

1. 打开 Log Interaction 节点，找出写错的两处字段映射，改成正确语义（intent 列 ← intent、userName 列 ← first_name），并给升级链补上 Prepare Log Data；
2. 故意把分类模型的 URL 改错，发一条消息，走一遍完整错误网：Slack 告警（内容？）→ 错误表（chatId？）→ 用户兜底（文案？）——用断言语义核对每一站；
3. 给 Send Reply to User 接一条 error 连线到 Handle API Error，重测"发送失败"场景，观察告警是否出现。