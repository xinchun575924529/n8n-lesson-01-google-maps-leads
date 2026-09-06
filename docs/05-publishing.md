# 入库 + 播报：从"发现"到"闭环"的最后两公里

## 结构地图

本段是全流程的"收口"环节，解决三个问题：**数据落库、防重闭环、人类可见的产出**。

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│ 评分筛选通过  │ ──▶ │ Build Report  │ ──▶ │  message     │ ──▶ Slack
│ 的 prospects │     │ (Code节点)    │     │  rows[]      │ ──┐
└─────────────┘     └──────────────┘     └─────────────┘  │
                                                            ▼
                                              ┌─────────────────────┐
                                              │ SplitOut → 逐行写     │
                                              │ Google Sheet         │
                                              │ (成为下轮去重名单)    │
                                              └─────────────────────┘
```

**核心设计哲学**：一次构建、两路分发——Slack 要"人话摘要"，Sheet 要"机器可读明细"。两者从同一个数据源派生，杜绝口径不一致。

---

## 1. Build Report：一份输入，两种形态

### 1.1 节点职责

这是一个 Code 节点，输入是前面筛选通过的 `prospects` 数组，输出是一个**元组**：

```js
// 输出结构（伪代码）
return {
  message: "Slack 播报的摘要文本（单条，不拆行）",
  rows: [/* 每个 prospect 一行，写 Sheet 用 */]
};
```

### 1.2 为什么是"一个节点出两份"而不是分开算？

**关键动机：保证一致性。** 如果 message 和 rows 由两个独立节点各自计算，未来改评分规则时很容易只改一处、漏掉另一处，导致 Slack 说"有 8 家"，Sheet 只写了 6 行。

### 1.3 rows 的字段设计

每一行对应一条 Sheet 记录，字段要**能支撑下次去重的全部判断**：

| 字段 | 用途 |
|------|------|
| `business_name` | 商家名（去重主键之一） |
| `address` / `city` | 地理去重 |
| `gap_score` | 本次评分，下次可追踪变化 |
| `gmap_url` | 直接跳转链接 |
| `reported_at` | **写当前时间戳**（90 天冷却窗口的起点） |

> 注意：Sheet 里每行都带 `reported_at`。下一次运行时，去重逻辑就是读这一列做 `Date.parse()` 判断是否在 90 天窗口内。

### 1.4 message 的组装逻辑

```js
// message 只取 Top 8（深度截断），拼成可读的几条
const top8 = prospects.slice(0, 8);
const lines = top8.map((p, i) => 
  `${i+1}. ${p.name}（${p.city}）— 缺口 ${p.gap_score} 分`
);
return {
  message: `本周发现 ${prospects.length} 家低评分商家，Top ${top8.length}：\n${lines.join('\n')}`,
  rows: prospects.map(p => ({ ..., reported_at: new Date().toISOString() }))
};
```

---

## 2. 急刹车：为什么 message 不拆行？

### 2.1 最自然的错误想法

初学 n8n 的人看到"8 家商家要通知"，第一反应是：

```js
// ❌ 错误示范
return prospects.map(p => ({
  message: `发现 ${p.name}，缺口 ${p.gap_score} 分`
}));
```

然后接一个 Loop 或直接连 Slack——**每条 prospect 推一条消息**。

### 2.2 灾难后果

这种情况如果 Slack 频道是 `#leads` 这样的共享频道，一周 8 条还算能忍；但一旦商家池扩大（比如扫 20 个城市），消息会变成 160 条/周——**频道直接变成垃圾场**，成员被迫静音或退出。而自动化系统的信任崩塌，往往就是这么一次"消息轰炸"。

### 2.3 正确姿势：一次播报，聚合摘要

```js
// ✅ 正确：全部合并成一条消息
return {
  message: `📊 本地商家商机周报（${new Date().toLocaleDateString()}）\n` +
    `共发现 ${prospects.length} 家低分商家，前 ${top8.length} 如下：\n` +
    lines.join('\n') +
    `\n\n完整名单已写入 Sheet：${sheetUrl}`
};
```

**原则**：Slack 是给人类"瞄一眼"的，不是数据管道。细节永远在 Sheet 里。

---

## 3. SplitOut → 逐行写 Sheet = 闭环去重

### 3.1 数据流

```
Build Report (rows[]) → SplitOut → 每个 item 一行 → Google Sheets 节点（Append）
```

### 3.2 为什么需要 SplitOut？

Google Sheets 节点的"Append"模式**接收的是数组**，它自己会拆行写入。那为什么还要一个显式的 SplitOut？

**两个原因**：

1. **可观测性**：SplitOut 之后每个 item 都带完整字段，在 n8n 编辑器里能直接看到"即将写入的每一行长什么样"，便于调试。
2. **可扩展性**：未来如果要在写 Sheet 之前对每一行做额外处理（比如补一个 UUID、加一列来源标记），SplitOut 后加个 Code 节点就行，不用重构 Build Report。

### 3.3 闭环含义

写入 Sheet 的每一行，在**下一次**工作流运行时，会被顶部的"读取已报备名单"节点读到，然后 `Date.parse(reported_at)` 判断 90 天窗口——**自己写出去的数据，就是自己下次去重的依据**。

这就是"闭环"：不需要额外维护状态数据库，Google Sheet 本身就是状态。

---

## 4. 坑点：top8 截断 ≠ 数据丢失

### 4.1 现象

Build Report 里写了 `prospects.slice(0, 8)`，新手常误以为"后面 9~20 名的商家被丢弃了"。

### 4.2 真相

**完全没有丢**。看代码：

```js
const top8 = prospects.slice(0, 8);   // 仅用于拼 message
rows: prospects.map(...)               // 全部商家都在 rows 里
```

`rows` 用的是完整数组 `prospects`，`slice` 只作用于 `message` 的展示层。

### 4.3 为什么只展示 8 条？

Slack 消息的**信息密度**问题。一条超过 20 行的 Slack 消息，阅读率直线下降。8 条正好是"扫一眼能记住"的上限。其余商家不是"不重要"，而是**完整名单在 Sheet 里**——渠道分工不同。

> 判断技巧：看到 `.slice(0, 8)` 但后续又用了原始数组，先确认是否只是"展示截断"。真正的数据丢弃，只会发生在 `.filter()` 且结果不再传递出去的场景。

---

## 5. 配置单一信源：URL 不硬编码

### 5.1 反模式

```js
// ❌ 在 Code 节点里硬编码
const SHEET_URL = "https://docs.google.com/spreadsheets/d/abc123...";
```

### 5.2 问题

1. 换 Sheet 要改 Code → 重部署 → 容易出错
2. 多个节点引用同一 Sheet 时，改一处漏一处
3. 别人看模板时不知道这个 URL 从哪来

### 5.3 n8n 的正确做法

用 **Sticky Note（便签）或 Set 节点作为配置中心**：

```
[配置节点] 存：{ sheetId, slackChannel, minGapScore }
     │
     ├──▶ Build Report（引用 sheetId 拼 URL）
     ├──▶ Google Sheets 节点（引用 sheetId）
     └──▶ Slack 节点（引用 slackChannel）
```

在 Code 节点里通过 `$json.sheetId` 读取，而不是写死：

```js
// ✅ 在 Build Report 里
const sheetUrl = `https://docs.google.com/spreadsheets/d/${$json.sheetId}`;
```

### 5.4 模板的示范

原模板把 Slack Channel、Sheet ID 都放在工作流开头的配置区（便签 + Set/Code），后续所有节点只读引用。**改配置不需要碰逻辑节点**——这是 n8n 工作流"可维护性"的分水岭。

---

## 6. 完整链路回顾（本段结尾）

```
评分通过 → Build Report (message + rows) 
                ├── rows → SplitOut → 每行写 Sheet（追加）
                │