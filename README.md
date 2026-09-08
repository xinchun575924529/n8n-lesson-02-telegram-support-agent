# n8n 实战课 · 用 Telegram + Groq + Sheets + Slack 构建语音/文本分级客服

> 一个真实的 40 节点模板，拆解成可消化、能动手、会变通的 6 段进阶课。
> 客户在 Telegram 上发语音、发文字、发火、报被盗——一条工作流全部接住：语音转写、情绪识别、危险消息转人工、普通消息按成本分级 AI 自动回，全程留痕。

---

## 学完你将掌握什么

### 技术点
1. **多模态消息接入与语音转写**：Telegram 触发器同时接语音 / 文本 / 照片三类消息，语音走「下载 → 修复文件名 → Whisper 转写」，并学会在错误上下文里用**四级回退链**找回用户 chatId。
2. **LLM 结构化输出与容错解析**：让模型按严格 JSON schema 返回意图与情绪，再用 Code 节点做防御式解析——清洗 markdown 围栏、垃圾输入静默回退、`!!` 布尔归一。把 LLM 当"不可信输入"处理。
3. **分级路由的成本工程**：同一套 OpenAI 兼容接口，按意图把消息路由到**快而便宜的 8B 模型**（FAQ）或**强而慢的 70B 模型**（安全 / 交易），学会"不是所有问题都值得用最强模型"。
4. **状态锁与幂等防骚扰**：用 Google Sheets 当锁表，升级人工后写 30 分钟窗口——同一个人反复发火只通知一次人工、只发一次安抚，不重复打扰、不重复留痕。
5. **日志留痕与错误网设计**：正常路径全量写日志，任何 LLM / 转写失败统一走「Slack 告警 + 错误表 + 用户兜底回复」三连。更重要的是——**本课附赠 4 个从模板里挖出的真实 bug**（含官方模板字段映射错误），教你用 mock 断言"照妖"模板。

### 业务点
1. **AI 客服分级处理全流程**：完整体验"多模态接入 → 理解情绪 → 风险转人工 + 安抚 → 常规问题分级自动答 → 全量留痕可审计"这一条能直接接私域客服单子的流水线。加密钱包、电商、SaaS 的工单机器人都是同一套骨架。

---

## 前置需要

| 类别 | 要求 |
|------|------|
| 账号 | Telegram Bot Token（BotFather 建，客服 + 通知各一个）/ Groq API Key（免费额度够跑） / Google Sheets（一张表三个 sheet） / Slack（可选，错误告警用，可换群机器人） |
| 时间 | 45 分钟（6 节课，每节约 7 分钟） |
| 版本 | n8n 1.x 及以上（本地 Docker 或云版均可） |
| 建议 | 对 Telegram 消息、HTTP Request、Code 节点有最基础认知即可；本课会带你读 JSON 和 JavaScript 片段 |

> 💡 **没有 Telegram / Slack 也能学**：本课附的验证方法全程用 mock 层（本地 JSON + 桩节点）跑通 42 项场景断言，凭证只在你真正上线时才会用到。

---

## 课程结构地图

```
n8n-lesson-02-telegram-support-agent/
├── README.md                  ← 你在这里
├── workflow.json              ← 可直接导入 n8n 的真实工作流（40 节点，官方原版）
├── docs/
│   ├── 00-overview.md         模板全局拆解（40 节点地图 + 五段主链 + 错误网 + 凭证映射）
│   ├── 01-intake.md           多模态接入：语音转写 / 文本直读 / 照片拒绝
│   ├── 02-classify.md         LLM 意图情绪分类 + 防御式 JSON 解析
│   ├── 03-escalation.md       升级人工：30 分钟防轰炸锁 + TTS 语音安抚（含 0 输出断链雷）
│   ├── 04-routing.md          意图路由 × 分级模型（8B / 70B 成本工程）
│   └── 05-delivery-errors.md  回复发送 + 日志留痕 + 错误网（含官方模板两个实证 bug）
├── script/
│   └── short-video.md         抖音 60s 口播稿 + 素材清单
├── exercises/
│   └── exercises.md           6 道练习（含参考答案）
└── site/
    └── index.html             GitHub Pages 静态预览站
```

每节文档统一结构：**先结论 → 再拆解 → 坑点提示 → 动手练习**。

## 立即动手：导入本工作流

1. 下载 [`workflow.json`](workflow.json)（或直接复制 [raw 链接](https://raw.githubusercontent.com/xinchun575924529/n8n-lesson-02-telegram-support-agent/main/workflow.json)）
2. n8n 界面右上角 → **Import from File**（或 CLI：`n8n import:workflow --input=workflow.json`）
3. 按 [`docs/00-overview.md`](docs/00-overview.md) 的凭证映射表补齐连接（Telegram ×2 / Groq / Google Sheets / Slack）
4. 给自己的 Telegram bot 发一条语音 → 见证全流程

## 快捷链接

- 模板公开页：[n8n.io/workflows/17522](https://n8n.io/workflows/17522/)
- 本机已验证：**8 个 Code 节点 32/32 单元断言全过 + 7 大场景 42/42 mock 全链断言全过**，并挖出 4 个模板真实缺陷（详见 docs/03 与 docs/05），可放心在此教案基础上二次开发。
- REST API 获取原始模板 JSON：`https://api.n8n.io/api/workflows/templates/17522`