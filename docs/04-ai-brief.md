# 教案04 · AI 读差评 + 写简报

## 结构地图

```
目标：让 GPT 读"差评原文" → 产出结构化诊断 + 证据式外联简报

① 差评输入准备        → 来自上游 Code：top_reviews(≤3星前5条)
② lmChatOpenAi ×2    → 两个LLM：一个提炼诊断，一个组装简报
③ informationExtractor → 结构化抽取：complaint_themes + biggest_complaint
④ 枚举归一化(坑1)     → Code节点：非法情绪枚举归到4档
⑤ 兼容读取(坑2)       → Code节点：处理嵌套 output
⑥ pitch 组装          → Code节点：证据式拼接，禁主观形容词
⑦ DeepSeek 替换方案   → 改 baseURL 或换节点类型
```

---

## 一、整体架构：3 个节点的分工

| 节点 | 类型 | 职责 |
|---|---|---|
| `lmChatOpenAi` (诊断) | Language Model | 将差评原文喂给模型，输出自由文本诊断 |
| `informationExtractor` | Information Extractor | 从诊断文本中结构化抽取字段（基于 schema） |
| `lmChatOpenAi` (简报) | Language Model | 基于结构化字段 + 门店信息，生成外联简报 |

**流程示意：**

```
top_reviews (差评原文，来自上游)
      ↓
lmChatOpenAi#1 — "请分析以下差评的投诉主题..."
      ↓ (模型返回诊断文本)
informationExtractor — schema: complaint_themes[] / biggest_complaint / has_complaints
      ↓ (JSON 输出)
Code 节点 — 枚举归一化 + 兼容读取
      ↓ (干净的 diagnostics)
lmChatOpenAi#2 — "请基于以下事实生成一封外联简报..."
      ↓
pitch 文本 → 下游写 Sheet / 发 Slack
```

---

## 二、informationExtractor 的关键 schema

这是全模板里唯一一个 `Information Extractor` 类型节点。它的价值在于：**用 schema 约束 LLM 输出格式**，避免自由文本无法程序化消费。

```json
{
  "schema": {
    "type": "object",
    "properties": {
      "complaint_themes": {
        "type": "array",
        "items": { "type": "string" },
        "description": "差评中反复出现的投诉主题，如 '糟糕的服务态度'、'等待时间过长'"
      },
      "biggest_complaint": {
        "type": "string",
        "description": "最主要的投诉点，一句话概括"
      },
      "has_complaints": {
        "type": "boolean",
        "description": "是否存在实质性投诉（非恶意或无内容评价）"
      }
    },
    "required": ["complaint_themes", "biggest_complaint", "has_complaints"]
  }
}
```

**触发方式**（在 n8n 的 Information Extractor 节点配置里）：

- **Input Column**：选上游 LLM 输出的列（如 `diagnosis_text`）
- **Schema**：粘贴上述 JSON
- **Model**：可用同一模型，也可以复用 `lmChatOpenAi` 节点

> 实际效果：模型输出会被强制解析成合法 JSON，schema 保证了字段永远存在。

---

## 三、坑点 1：模型输出非法枚举 → 正则归一到 4 档

**现象**：模板中某个 Code 节点会对模型输出的情绪/态度做枚举分类。模型可能输出：

- `"very negative"`（不是预设枚举值）
- 或者夹杂了标点、大小写变体

**预设的 4 档情绪枚举**（模板里的常量）：

```js
const VALID_TONES = [
  'mixed',            // 混合 - 有褒有贬
  'mostly negative',  // 多数负面
  'mostly positive',  // 多数正面
  'unclear'           // 模糊不清
];
```

**归一化逻辑（模板实测通过断言）**：

```js
// 模型输出的原文例如："Very Negative!"
let rawTone = $json.tone_from_model || '';

// 1. 统一小写、去标点
rawTone = rawTone.toLowerCase().replace(/[^a-z ]/g, '').trim();

// 2. 关键词匹配，归一到 4 档
let normalizedTone;
if (/negative|bad|poor|terrible/.test(rawTone)) {
  normalizedTone = 'mostly negative';
} else if (/positive|good|great|excellent/.test(rawTone)) {
  normalizedTone = 'mostly positive';
} else if (/mix|neutral|both/.test(rawTone)) {
  normalizedTone = 'mixed';
} else {
  normalizedTone = 'unclear';  // 兜底
}

// 3. 保证永远不脏
return { ...$json, tone_normalized: normalizedTone };
```

**坑点本质**：LLM 是概率模型，不保证输出匹配 schema 的枚举约束。正则兜底是最后防线，**绝不能假设模型会乖乖听话**。

---

## 四、坑点 2：extractor 输出可能嵌套在 output 之下

**现象**：Information Extractor 的返回结构不统一。某些版本/模型下，输出形如：

```json
{
  "output": {
    "complaint_themes": ["服务差", "排队久"],
    "biggest_complaint": "排队时间过长",
    "has_complaints": true
  }
}
```

也可能直接平铺在顶层。如果在 Code 里写死 `$json.complaint_themes`，一旦遇到嵌套结构就会 `undefined` → 后续 join 报错。

**兼容读取写法（模板实测通过）**：

```js
// 安全读取，兼容两种形态
const raw = $json.output || $json;   // 若有 output 键，取它；否则取自身

const themes = Array.isArray(raw.complaint_themes)
  ? raw.complaint_themes
  : [];

const biggest = raw.biggest_complaint || '';
const hasComplaints = !!raw.has_complaints;

// 再补一道类型防线
const themesText = themes
  .filter(t => typeof t === 'string' && t.trim())
  .join('、');

return {
  themesText,
  biggest,
  hasComplaints
};
```

> 模板原作者的写法更"防御"：先 `typeof $json.output === 'object'` 判断再取值。你可以比照这个思路，但上面这种 `|| ` 兜底更简洁。

---

## 五、pitch 的结构：纯证据式，禁主观形容词

模板中生成外联简报时有一个硬性规则：**只摆事实，不用"很好""不堪""糟糕"这类主观词**。

**目标句式（证据式）**：

```
[店名]：目前 Google 档案存在[证据1]，和[证据2]。
Reviewers mention [主题1] 和 [主题2]。
```

**具体实现**（简化版 Code）：

```js
const businessName = $json.business_name;
const evidence1 = $json.missing_website ? '未认领 Google 商家档案' : '官网链接缺失';
const evidence2 = $json.peer_rating_gap > 0.3 ? `评分低于同行 ${(peer_rating_gap).toFixed(1)} 分` : `评论量仅为同行中位数的 ${pct}%`;

const themesText = $json.themesText || '';  // 前面归一化后的

// 拼装：只用名词短语和事实，不掺杂形容词
const pitch =
  `${businessName}：缺少${evidence1}，和${evidence2}。` +
  `Reviewers mention ${themesText}.`;

return { pitch };
```

**对照**（违反规则的写法）：

```
❌ "贵店评分太低了，服务质量令人失望"  ← 有主观情绪，容易引发抵触
✅ "贵店当前未认领 Google 档案，且评分低于同行均值 0.7 分"  ← 可核实、不可反驳
```

**设计原因**：外联简报复用给潜在客户，对方看到的是"客观诊断"而非"指责"。证据式文案更容易打开对话窗口。

---

## 六、DeepSeek 替换 OpenAI 的两种方案

> 模板里写的是 `gpt-5.6-terra`（实际不存在），必须替换成可用模型。DeepSeek 兼容 OpenAI API，直接换 baseURL 即可。

### 方案 A：修改原 `lmChatOpenAi` 节点（推荐）

在 **OpenAI 节点** 的 Credential / Base URL 处覆盖：

```
Base URL: https://api.deepseek.com/v1
Model:    deepseek-chat
```

**n8n 具体步骤：**

1. 双击任意 `lmChatOpenAi` 节点
2. 在 **Model** 输入框里填：`deepseek-chat`
3. 在 **Base URL** 覆盖为：`https://api.deepseek.com/v1`
4. API Key 换成 DeepSeek 控制台申请的 key
5. 如果有多个 lmChatOpenAi 节点，确认全部改完

### 方案 B：换成 `deepSeekApi` 节点类型

如果你 n8n 装了 DeepSeek 的社区节点包（`n8n-nodes-langchain.deepseek`），可以直接：

1. 删除原 `lmChatOpenAi`
2. 拖入 `DeepSeek` 类型（或 `deepSeekApi`）
3. 配置模型：`deepseek-chat` 或 `deepseek-reasoner`
4. 连接方式保持与上游/下游不变

> 注意：换节点类型后，字段名（如 `model` 的传参方式）可能有细微差异，改完必须跑一次测试。

---

## 七、全链路串联示意（4 号节点后的输出结构）

经过 AI 简报模块后，传给下游的完整字段：

```json
{
  "business_name": "某咖啡馆",
  "pitch": "某咖啡馆：缺少认领 Google 档案，和评分低于同行 0.7 分。Reviewers mention 服务慢 和 空调不冷.",
  "complaint_themes": ["服务慢", "空调不冷"],
  "biggest_complaint": "服务慢",
  "has_complaints": true,
  "tone_normalized": "mostly negative",
  "window_start": "2024-03-01",
  "window_end": "2024-05-30"
}
```

下游（Sheet 行写入 + Slack 消息拼装）只消费 `pitch` 一个字符串，其他字段作为调试/归档用。

---

## 坑点总结表

| # | 坑点 | 现象 | 解法 |
|---|---|---|---|
| 1 | LLM 输出非法枚举 | 模型吐出 `very negative` 而非预设值 | 正则关键词匹配 → 归一 4 档 → 兜底 `unclear` |
| 2 | extractor 输出嵌套 | `$json.output` 不是顶层 | `const raw = $json.output \|\| $json` 兼容 |
| 3 | 模型名称不可用 | 模板写 `gpt-5.6-terra` 不存在 | 替换为 `deepseek-chat` / `gpt-4o-mini` 等 |
| 4 | 简报含主观词 | 模型自行加"十分糟糕""强烈推荐" | 代码层把模型输出当作"素材"，拼装模板由 Code 控制 |