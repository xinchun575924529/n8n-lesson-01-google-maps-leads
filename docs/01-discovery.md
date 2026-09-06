# 第1段 · Bright Data 异步发现

## 结构地图

```
本段目标：用 Bright Data 的 Dataset API 找到"某个城市里某个品类的商家列表"
│
├─ 整体模式：异步四步（不是一次 HTTP 拿结果）
│
├─ 第一步：trigger（提交任务）→ 拿到 snapshot_id
│
├─ 第二步：轮询 progress（查进度）→ 等状态变 ready
│
├─ 第三步：下载 snapshot（拿数据）→ 得到商家 JSON 数组
│
└─ 第四步：解析数据（本模板用 Code 节点做防御式校验）
```

### 核心结论（先记住这三条）

1. **Bright Data 的 Web Unlocker / Dataset API 是异步任务模式**：你提交一个"抓取任务"，它给你一个 `snapshot_id`，你需要主动轮询进度，等它跑完再下载结果。不是发一个请求就同步返回数据。
2. **整个过程上限约 7 分钟**：模板里 `MAX_POLLS = 20`，每次间隔 20 秒，`20 × 20s = 400s ≈ 6.7 分钟`。超过就放弃，进错误分支。
3. **HTTP 200 不代表成功**：Bright Data 经常在业务层面报错时仍然返回 HTTP 200，错误信息藏在 body 里。所以第一步必须同时检查 `error` 字段和 `snapshot_id` 字段。

---

## 第一步：Trigger（提交抓取任务）

### 请求示例

**URL**（POST）：

```
https://api.brightdata.com/datasets/v3/trigger
```

**Headers**：

```
Authorization: Bearer <你的_BRIGHT_DATA_API_KEY>
Content-Type: application/json
```

**Body**：

```json
{
  "dataset_id": "gd_m8ebnr0q2qlklc02fz",
  "payload": [
    {
      "keyword": "locksmith Brooklyn",   // 品类 + 城市，必须写在一起，没有独立的 location 字段
      "country": "US"
    }
  ]
}
```

> **注意**：`dataset_id` 是 Bright Data 预置的"Google Maps 商家数据"模板 ID，**不是你自己创建的**。这个 `gd_m8ebnr0q2qlklc02fz` 是模板作者绑定好的，实测有效。

### 响应示例

```json
{
  "snapshot_id": "s_m9h4k2j8d7f6a1b3c5e9g0",
  "status": "running"
}
```

### 拿到什么

一个 `snapshot_id`，它是后续所有操作的凭证。**这个 ID 要传给第二个节点**。

---

## 第二步：轮询 Progress（查任务进度）

### 为什么不能立刻下载？

因为 Bright Data 需要时间去 Google Maps 上搜索、渲染、抓取、清洗数据。`trigger` 返回时任务刚启动，数据还没准备好。

### 请求示例

**URL**（GET）：

```
https://api.brightdata.com/datasets/v3/progress/{snapshot_id}
```

**Headers**：

```
Authorization: Bearer <你的_BRIGHT_DATA_API_KEY>
```

**响应示例**（三种状态）：

```json
// 情况1：还在跑
{ "status": "running", "progress": 42 }

// 情况2：失败（比如 keyword 非法、配额不够）
{ "status": "failed", "error": "Dataset not found" }

// 情况3：完成，可以下载了
{ "status": "ready", "progress": 100 }
```

### 模板里的轮询逻辑（伪代码）

```js
const MAX_POLLS = 20;
const POLL_INTERVAL_MS = 20 * 1000;   // 20秒

for (let i = 0; i < MAX_POLLS; i++) {
  const response = await fetch(progressUrl);
  const data = await response.json();

  if (data.status === 'ready') {
    return { snapshotId, ready: true };
  }
  if (data.status === 'failed') {
    return { snapshotId, ready: false, error: data.error };
  }
  // running → 等 20 秒再试
  await sleep(POLL_INTERVAL_MS);
}

// 20次都没 ready
return { snapshotId, ready: false, error: `Timeout after ${MAX_POLLS} polls` };
```

### MAX_POLLS 20 × 20s = 7 分钟的取舍

| 参数 | 值 | 理由 |
|------|-----|------|
| MAX_POLLS | 20 | 大多数 Bright Data snapshot 在 2~5 分钟内完成，20 次足够覆盖绝大多数任务 |
| 间隔 | 20s | 轮询太频繁浪费配额（API 按请求计费），太慢让用户等太久 |
| 总预算 | ~7 分钟 | 超过 7 分钟说明任务异常（dataset_id 错、配额耗尽、网络问题等），与其死等不如走错误分支 |

> **坑点提示**：如果你把 `MAX_POLLS` 调大（比如 50），你的 n8n 工作流在等待时会一直占用执行线程（如果是单线程模式会阻塞其他任务的执行）。模板作者用 20 是权衡了成功率和资源占用后的经验值。

---

## 第三步：下载 Snapshot（拿数据）

### 请求示例

**URL**（GET）：

```
https://api.brightdata.com/datasets/v3/snapshot/{snapshot_id}
```

**Headers**：

```
Authorization: Bearer <你的_BRIGHT_DATA_API_KEY>
```

**响应示例**（这是一个 JSON 数组，每个元素是一个商家的记录）：

```json
[
  {
    "name": "ABC Locksmith Brooklyn",
    "address": "123 Main St, Brooklyn, NY",
    "rating": 4.2,
    "reviews": 37,
    "website": "https://abclocks.com",
    "claimed": false,
    "people_also_search": [
      { "name": "XYZ Locksmith", "rating": 4.5, "reviews": 120 }
    ],
    "top_reviews": [
      { "text": "They never showed up!", "rating": 1 },
      { "text": "Fast service, fair price", "rating": 5 }
    ],
    "category": "Locksmith",
    "hours": { "monday": "8:00-18:00" },
    "photos_count": 4
  }
]
```

### 拿到什么

一个包含完整商家字段的数组，后面所有节点（评分卡、对标签、差评分析）都从这个数组里取数据。

---

## 第四步：解析与防御式检查（Code 节点内）

模板作者在这个环节做了非常细致的防御式校验，教学价值极高。核心逻辑：

```js
// 伪代码：检查 response 里是否有 error 字段（即使 HTTP 200）
if (response.body.error) {
  throw new Error(`Bright Data API error: ${response.body.error}`);
}

// 检查是否真的拿到了 snapshot_id
if (!response.body.snapshot_id) {
  throw new Error('Missing snapshot_id in trigger response');
}

// 检查下载的数据是否真的是数组
if (!Array.isArray(snapshotData)) {
  throw new Error('Snapshot data is not an array');
}

// 逐条检查每条记录是否有关键字段
const requiredFields = ['name', 'rating', 'reviews'];
for (const record of snapshotData) {
  for (const field of requiredFields) {
    if (record[field] === undefined || record[field] === null) {
      throw new Error(`Record missing required field: ${field}`);
    }
  }
}
```

---

## 坑点 1：Bright Data 用 HTTP 200 返回错误

### 现象

发送 trigger 请求后，HTTP 状态码是 200，但 body 里是：

```json
{
  "error": "Dataset not found or access denied",
  "status": "failed"
}
```

### 原因

Bright Data 的 API 设计是**业务错误不改变 HTTP 状态码**。它的错误处理策略是"请求成功到达服务器"用 200 表示，至于业务逻辑是否成功，看 body 里的 `error` 字段。

### 为什么必须先检查 body 里的 error / snapshot_id

如果不检查，你会：

1. 拿着空的 `snapshot_id` 去轮询 `progress`（或拿着 `undefined` 拼 URL）
2. 得到 404 或永远 `running`，浪费 7 分钟轮询预算
3. 排查问题时很难定位——n8n 日志只显示 HTTP 200，看起来一切正常

### 正确姿势

拿到 trigger 响应后，**第一件事**就是检查：

```js
const data = response.body;

// 两个条件都要检查
if (data.error) {
  throw new Error(`Bright Data error: ${data.error}`);
}
if (!data.snapshot_id) {
  throw new Error('Missing snapshot_id — trigger failed silently');
}
```

> **坑点提示**：`data.error` 可能是字符串（`"error": "Dataset not found"`）也可能是对象（`"error": {"code": 40001, "message": "..."}`）。防御式写法建议统一处理：`typeof data.error === 'string' ? data.error : JSON.stringify(data.error)`。

---

## 坑点 2：keyword 必须包含城市名（没有 location 字段）

### 现象

很多人刚接触 Bright Data Google Maps dataset 时，会自然而然地想：既然有 `country` 字段，那我来个 `"city": "Brooklyn"` 不就行了？—— **不行**。

### 原因

这个 `gd_m8ebnr0q2qlklc02fz` 数据集的 schema 里**只有两个字段**：

| 字段 | 说明 |
|------|------|
| `keyword` | 完整搜索词，Bright Data 直接在 Google Maps 里搜这个词 |
| `country` | ISO 国家码（如 US、GB、DE），用于限定搜索地区 |

它没有 `city`、`state`、`lat/lon` 这样的独立字段。**Bright Data 的搜索逻辑就是"我拿 keyword 去 Google Maps 搜索框里打一遍，然后把结果抓回来"**。

所以如果你 keyword 只写 `"locksmith"`，它会把整个美国的 locksmith 全抓回来（甚至跨国家，取决于 country 和 Google 的本地化策略）。你的后续评分、去重、外联全都乱套。

### 正确姿势

把城市名直接拼进 keyword：

```
// 错误 ❌
{ "keyword": "locksmith", "country": "US" }

// 正确 ✅
{ "keyword": "locksmith Brooklyn", "country": "US" }