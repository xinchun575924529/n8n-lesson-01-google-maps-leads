# n8n 实战课 · 用 Bright Data + GPT + Sheets + Slack 构建本地商家线索挖掘机

> 一个真实的 32 节点模板，拆解成可消化、能动手、会变通的 6 段进阶课。

---

## 学完你将掌握什么

### 技术点
1. **异步任务编排**：掌握 Bright Data 异步数据管道的完整闭环（触发 → 轮询 → 下载 → 解析），并把这种模式复制到任何「提交任务 → 等待结果」的 API 场景。
2. **防御式 Code 节点设计**：学会在 n8n Code 节点里做 31 项单元断言级别的防御检查，识别 HTTP 200 但 body 带 error 的"假成功"响应，写出比业务逻辑还健壮的解析代码。
3. **评分卡与数据建模**：把一个模糊的"商家档案是否优质"拆解成 8 项可量化的评分维度和同行对标逻辑，构建可解释、可调参的商业打分模型。
4. **多应用数据流拼接**：打通 Sheets（去重/存档）→ GPT（内容生成）→ Slack（通知）三端，并解决日期解析失败、排序后数据漂移、模型输出不确定等实际工程问题。

### 业务点
1. **本地商家拉新全流程**：完整体验"发现线索 → 质量评分 → 去重冷却 → AI 外联文案 → 入档通知"这一条能直接拿去接单或自用的业务流水线。

---

## 前置需要

| 类别 | 要求 |
|------|------|
| 账号 | Bright Data API Key / Google Sheets OAuth / Slack Bot Token / OpenAI 或 DeepSeek API Key |
| 时间 | 45 分钟（6 节课，每节约 7 分钟） |
| 版本 | n8n 1.x 及以上（本地 Docker 或云版均可） |
| 建议 | 对 HTTP Request 节点和 Code 节点有最基础的认知即可，不需要精通 JavaScript |

---

## 课程结构地图

```
lesson-01-google-maps-leads/
├── README.md                  ← 你在这里
├── workflow.json              ← 可直接导入 n8n 的真实工作流（32 节点）
├── docs/
│   ├── 00-overview.md         模板全局拆解（32 节点地图 + 数据流向总览 + 凭证映射）
│   ├── 01-discovery.md        Bright Data 异步发现（触发 → 快照轮询 → 下载）
│   ├── 02-scoring.md          缺口评分卡（8 维打分规则 + 同行对标）
│   ├── 03-dedupe.md           去重 + 90 天冷却 + "安静分支"
│   ├── 04-ai-brief.md         LangChain 读差评 + 证据式简报 + DeepSeek 换底
│   └── 05-publishing.md       入库 + Slack 播报（top8 截断与单源配置）
├── script/
│   └── short-video.md         抖音 60s 口播稿 + 素材清单
├── exercises/
│   └── exercises.md           6 道练习（含参考答案）
└── site/
    └── index.html             GitHub Pages 静态预览站
```

每节文档统一结构：**先结论 → 再拆解 → 坑点提示 → 动手练习**。

## 立即动手：导入本工作流

1. 下载 [`workflow.json`](workflow.json)（或直接复制 [raw 链接](https://raw.githubusercontent.com/xinchun575924529/n8n-lesson-01-google-maps-leads/main/workflow.json)）
2. n8n 界面右上角 → **Import from File**（或 CLI：`n8n import:workflow --input=workflow.json`）
3. 按 [`docs/00-overview.md`](docs/00-overview.md) 的凭证映射表补齐 4 个连接（Bright Data / Google Sheets / Slack / OpenAI·可换 DeepSeek）
4. 修改 **Set Search Config** 里的 `city` 和 `categories` 即可跑起来

## 快捷链接

- 模板公开页：[n8n.io/workflows/18238](https://n8n.io/workflows/18238/)
- 本机已验证：**31/31 单元断言全部通过**（7 个 Code 节点），可放心在此教案基础上二次开发。
- REST API 获取原始模板 JSON：`https://api.n8n.io/api/workflows/templates/18238`