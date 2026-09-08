# 教案02 · 练习题集

> **建议做题顺序**：第 1 题（通读结构）→ 第 2 题（抓容错设计）→ 第 3 题（改锁逻辑）→ 第 4 题（加新意图）→ 第 5 题（修官方 bug）→ 第 6 题（补错误网盲区）。前三题训练"看懂模板"，后三题训练"动手改造 + 验证"。

---

## 第 1 题 · 主干流程与三岔分流

**题干**：仅凭教案 docs/00 与 docs/01 的事实包，回答：
1. 该模板 32 个功能节点可以分成哪五段主链 + 一张什么网？每段一句话概括职责；
2. 一条 Telegram 语音消息从进来到发出回复，依次经过哪些节点（写节点名即可）？
3. 为什么照片消息走"Unsupported 回复"而不是丢给 LLM 处理？模板的回复策略给了什么启发？

**参考答案**

<details>
<summary>点击展开</summary>

1. **五段主链 + 一张错误网**：① Intake 接收（多模态转成统一 transcript）→ ② Classify 理解（LLM 分类 intent/sentiment/security_risk）→ ③ Escalation 升级（危险消息转人工 + 30 分钟防轰炸锁 + 语音安抚）→ ④ Routing 路由（switch 按 intent 分档模型）→ ⑤ Delivery 交付（发送回复 + 组装日志 + 写 Interactions 表）。外加一张错误兜底网（Handle API Error → Slack 告警 + 错误表 + Fallback 回复），罩住转写/分类/回复生成链。

2. `Voice Message Trigger → Has Voice?（是）→ Get Voice File → Fix Audio Filename → Transcribe (Whisper) → Extract Transcript (Voice) → Classify Intent & Sentiment → Parse Classification JSON → Get row(s) in sheet → Evaluate Escalation Lock → Already Escalated?（否）→ Escalation Check（假）→ Route by Intent →（假设 faq）→ Fast Model Reply (FAQ) → Extract Reply Text → Send Reply to User → Prepare Log Data → Log Interaction`

3. 模板遵循"宁可不答，不要瞎答"：照片/贴纸等非语音非文本内容不在设计范围内，硬答（让 LLM 看图）会引入不可控行为，拒绝并引导用户"打字描述"反而体验最稳。启发：**客服 bot 的回复策略要管理用户期望**——明确告知边界比含糊其辞更专业。

</details>

---

## 第 2 题 · 防御式 JSON 解析四件套

**题干**：`Parse Classification JSON` 把 LLM 输出当"不可信输入"。请回答：
1. 模型可能返回哪 4 种脏输出？各自被哪道防御接住？
2. 解析彻底失败时，为什么回退值是 `faq/neutral/false/0` 而不是抛错或回退成别的？
3. `security_risk` 的 `!!` 双非归一解决了什么问题？给出 `"true"`、`1`、`"false"`、`true` 四种输入各自的输出。

**参考答案**

<details>
<summary>点击展开</summary>

1. ① markdown 围栏（```json ... ```）→ 围栏清洗后 parse；② 前后夹废话（"Here is your JSON: {...}"）→ 提取第一段 `{...}`；③ 布尔写成字符串（`"security_risk":"true"`）→ `!!` 真值归一；④ 完全不是 JSON → 静默回退默认档。
2. 因为**"宁可答错方向，不能没有回复"**：faq 是 switch 里必然存在的最安全分支（不会升级人工、不会触发强模型烧钱），neutral/false 不会误升级。如果回退成抛错，整条客服链会断，用户被晾着——比答偏更伤信任。
3. 解决"模型不守 schema、把布尔当字符串/数字返回"导致的类型错乱。`"true"`→`true`、`1`→`true`、`"false"`→`false`（注意 `!!"false"` 在 JS 里是 `true`，所以实现要先比较字符串内容再取反，不能直接 `!!`！）、`true`→`true`。
</details>

---

## 第 3 题 · 30 分钟防轰炸锁

**题干**：锁机制是"查 → 判 → 写"三段式。请回答：
1. 为什么必须先查锁再升级，而不是升级完再写锁？
2. `Evaluate Escalation Lock` 对"escalatedAt 缺失"的行判未锁，是 fail-open 还是 fail-closed？为什么这样设计合理？
3. 动手：把锁窗口从 30 分钟改成 5 分钟，写一个 6 分钟前的锁行，发一条 angry 消息——预期发生什么？

**参考答案**

<details>
<summary>点击展开</summary>

1. 锁的目的是"同一个 chatId 在窗口内只升级一次"。如果先升级再写锁，第一次消息就漏判了（查的时候还没有锁）——第二次才会被拦。先查后写才能让**第一次升级正常执行、第二次起被拦截**。
2. **fail-open**（放宽）：字段缺失按未锁处理，允许再次升级。因为升级的代价是"多打扰一次人工"（成本问题），而漏升级的代价是"安全事件没人管"（事故问题）——成本问题倾向放行。
3. 6 分钟 > 5 分钟窗口 → 锁已过期 → `Evaluate Escalation Lock` 判未锁 → 走 Escalation Check（angry=true）→ 再次升级：写新锁、通知人工、发语音安抚。若锁行是 3 分钟前的，则判锁中 → Holding Reply，不通知人工。
</details>

---

## 第 4 题 · 给客服加一个新意图 refund

**题干**：老板要求支持"退款"意图（用户问怎么退款）。请按模板的套路列出**需要同步修改的全部位置**，并说明漏改任一处的后果：

1. 分类节点的 system prompt；
2. Parse Classification JSON 的回退表；
3. Route by Intent（switch）；
4. 路由目标（接哪档模型）；
5. 日志/回复文案（可选）。

**参考答案**

<details>
<summary>点击展开</summary>

1. **分类 system prompt**：schema 的 intent 枚举加 `refund`，并给一两句判据（"涉及申请退款/退回资金"）。漏改 → 模型永远不会输出 refund；
2. **Parse 回退表**：若希望垃圾输入回退到 refund 以外的安全档，检查默认回退仍是 faq（建议保持 faq，refund 不应是默认猜测）。漏改 → 无直接影响，但要意识到回退值应指向"最安全出口"；
3. **Route by Intent（switch）**：加 `refund` 分支。⚠️ **漏改 = 头号大坑复发**：模型一旦输出 refund，switch 无分支命中 → 0 项输出 → 发送节点静默不跑，用户消息石沉大海！
4. **路由目标**：refund 建议接 Contextual 70B（涉及具体订单上下文），像 transaction_issue 一样处理。漏改 → 接错档位（如接了 8B FAQ，退款政策这种高价值问题答太浅）；
5. **日志/文案**：如果 refund 需要特殊回复模板或日志标记，同步加。漏改 → 日志无法区分退款咨询，报表失真。

**口诀**：加一个意图 = 改 schema → 查回退 → 加 switch → 定档位 → 看日志，五处联动，漏一处就是一次静默吞消息。
</details>

---

## 第 5 题 · 修官方模板的 Log Interaction bug

**题干**（本课最实战的一题）：模板的 `Log Interaction` 节点存在列映射错位——`intent` 列存的是 sentiment 值、`userName` 列存的是 transcript。升级链还跳过了 `Prepare Log Data`。请：

1. 打开 Log Interaction 节点，找出两处写错的字段表达式，改为正确语义（intent 列 ← intent、userName 列 ← first_name 或 chatId）；
2. 把升级链 `Send Voice Reply to User → Log Interaction` 改成 `Send Voice Reply to User → Prepare Log Data → Log Interaction`（与正常路径共用组装节点）；
3. 改完后，用"mock 场景断言"验证：发一条 wallet_security 消息（会升级），断言 Interactions 表出现一行 `escalated=true` 且 `intent=wallet_security`。

**参考答案**

<details>
<summary>点击展开</summary>

1. 在 Log Interaction 节点的字段映射区把表达式改成：`intent` 列 ← `={{ $json.intent }}`（若上游经 Prepare Log Data，则是组装好的字段名）、`userName` 列 ← `={{ $json.first_name }}`。改前先确认上游实际输出的字段名（这就是"列级断言"的价值——你得先知道规格是什么）；
2. 在 n8n 编辑器里：断开 `Send Voice Reply to User` 到 `Log Interaction` 的连线 → 接到 `Prepare Log Data` → 再把 `Prepare Log Data` 连到 `Log Interaction`。升级链从此与正常链共用组装逻辑，escalated=true 会正确写入；
3. 验证断言应包含：日志行存在（写入成功）+ `escalated == true`（升级链有标记）+ `intent == "wallet_security"`（列映射正确）。三条都过才算修完——只验第一条，Bug A 就会漏网。
</details>

---

## 第 6 题 · 补错误网盲区（发送节点裸奔）

**题干**：模板错误网只罩转写/分类/回复生成链，四个 Telegram 发送节点（Send Reply / Holding Reply / Notify Human Agent / Send Voice Reply）都没有错误分支。请设计改造方案：

1. 在 n8n 里给 `Send Reply to User` 挂一条 error 连线到 `Handle API Error`（节点设置 → On Error → Continue，或直接拖错误连线）；
2. 思考：`Handle API Error` 的 chatId 回退链在"发送节点自己失败"的场景下还能找到 chatId 吗？（提示：发送节点失败时，它的上游上下文还在不在？）
3. 扩展：如果发送失败是因为 bot 被用户拉黑，Fallback Reply 也发不出去——这种情况该怎么告警？（提示：错误网三连里哪一连仍然有效？）

**参考答案**

<details>
<summary>点击展开</summary>

1. 实操：把 `Send Reply to User` 的 error 输出（节点下方第二输出点，红色）连到 `Handle API Error`。之后发送失败会走错误网：Slack 告警 + 错误表 + 尝试 Fallback；
2. **能找到**：发送节点失败时，它**接收到的上游数据仍在错误上下文里**（n8n 的 error 输出携带该节点的输入），chatId 回退链第一级（Parse Classification JSON）可能已不在链上，但 Extrace Transcript / Trigger 层级的 chatId 通常还在——回退链设计恰好覆盖这种"越往后越难找"的场景；
3. bot 被拉黑时 Fallback 同样发不出去，但 **Slack 告警仍然有效**（那是内部通道，不依赖用户侧 bot）。所以错误网三连的告警是最后保险——这也是为什么"告警通道必须和用户通道物理隔离"（模板用第 2 个内部 bot + Slack 双通道，正是这个道理）。
</details>

---

## 附加题 · 验证方法论迁移

把本课"32 单测 + 42 场景断言"的方法用到你自己的模板上，写出三步操作清单（参考 docs/05 第 6 节）：① 字段规格表；② 列级断言；③ 边界审查（空输出 + 错误输出）。做完你会发现：**大多数模板的隐藏 bug，都藏在"能跑通"和"字段对"之间的缝隙里。**