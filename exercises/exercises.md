# 教案01 · 练习题集

> **建议做题顺序**：第 1 题（通读结构）→ 第 2 题（抓主干）→ 第 3 题（找隐藏逻辑）→ 第 4 题（修 bug）→ 第 5 题（改代码）→ 第 6 题（综合设计）。前三题训练"看懂模板"，后两题训练"手改逻辑"，最后一题训练"迁移复用"。

---

## 结构题（3 道）

### 第 1 题 · 主干流程图解

**题干**：不看 n8n 编辑器，仅凭教案中的事实包，画出该模板 24 个功能节点的"主干数据流"（不需要画出便签节点）。要求：
1. 按时间顺序列出主要阶段（建议 4-6 个）；
2. 在每个阶段标注**关键的输入/输出数据字段**；
3. 用箭头表示节点间依赖关系，文字描述即可（如 "A → B"）。

**参考答案**

<details>
<summary>点击展开</summary>

**阶段一 · 定时触发 / 参数注入**
`Schedule Trigger (周日07:00)` → `Code 节点`（拼装查询参数：keyword = 品类+城市, country）

**阶段二 · Bright Data 异步抓取**
`HTTP Request (POST /trigger)` → 获 `snapshot_id` → `Code 轮询节点`（每 20s × 最多 20 次，POST /progress/{id}）→ 状态 = `ready` 后 `HTTP GET /snapshot/{id}` 拿全量商家 JSON。

**阶段三 · 本地评分 + 同行对标**
`Code 评分卡节点`：
- 输入：商家原始字段（rating, review_count, website, claimed 等）
- 逐条计算 gap_score（25/20/15/15/12/8/10/5/5 分制）
- 对比 `people_also_search` 中同行中位数（peer_rating / peer_reviews）
- 过滤 gap_score < min_gap_score 的弱商家

**阶段四 · 冷却去重 + 差评抽取**
`Code 去重节点`：读 Google Sheet 已报备名单 → Date.parse 比对 90 天窗口 → 剔除已骚扰商家。
`Code 差评过滤节点`：top_reviews 中只保留 ≤3 星前 5 条，作为后续 GPT 阅读材料。

**阶段五 · AI 生成简报**
`LangChain (InformationExtractor)` → 提取 `complaint_themes` / `has_complaints` / 最大投诉
`GPT 节点`（DeepSeek，纯证据式文案）→ 输出 `pitch` 字符串

**阶段六 · 结果写入 + 通知**
`Code 组装节点`：产出 `(message, rows)` 元组，rows 截断 top 8
`Google Sheets 节点`：prospects 逐行写入 + 同时回写 sync 名单
`Slack 节点`：发送一次性汇总 message

</details>

---

### 第 2 题 · 状态机与分支识别

**题干**：该模板中有一个"轮询状态机"，请回答：
1. 轮询的**退出条件**有哪些？（成功/失败/超时分别如何判定）
2. 作者在 HTTP 200 响应的处理上做了一个特殊防御，是什么？为什么需要？（结合事实包第 2 条）
3. 除了轮询，模板中还有哪一处也是"分支逻辑"（if/else 性质）？

**参考答案**

<details>
<summary>点击展开</summary>

1. 退出条件：
   - **成功**：轮询接口返回 `status = ready`，此时立即 GET /snapshot/{id} 下载数据；
   - **失败**：返回 `status = failed`，直接抛错终止流程；
   - **超时**：`MAX_POLLS = 20` 次 × 20s 间隔 = 约 6.7 分钟仍未 ready，强制终止并告警。

2. 防御逻辑：**即使 HTTP 状态码为 200，也要检查响应 body 里是否内嵌 `error` 字段**。原因：Bright Data 官方 API 在业务异常时（如余额不足、dataset_id 失效）仍然返回 HTTP 200，但 body 中携带 error 对象。若只判 `response.status === 200` 就继续轮询，会把无效 snapshot_id 带入后续请求，造成死循环或脏数据。

3. 另一处分支：**Google Sheet 冷却去重节点**——对每一条已报备记录用 `Date.parse()` 解析日期，解析失败（NaN）时默认按"已报备"处理（防止重复骚扰）。这就是一个典型的 if/else 分支。

</details>

---

### 第 3 题 · 评分卡权重还原

**题干**：评分卡总分为 100 分（25+20+15+15+12+8+10+5+5）。现有一个真实商家数据如下，请计算它的 gap_score 并判断是否低于阈值（假设 min_gap_score = 30）：

| 字段 | 值 |
|---|---|
| claimed | false |
| website | 空 |
| rating | 3.8（同行中位数 4.2） |
| review_count | 12（同行中位数 58） |
| 差评（≤3星）占比 | 18% |
| photo_count | 5 |
| hours | 有 |
| 主营品类数 | 3 |
| service_options | 有 |

要求：
1. 逐项列出命中哪些扣分项；
2. 给出总分，并判断是否进入"外联候选名单"；
3. 指出哪一条扣分项最容易被人忽视但通常权重很高。

**参考答案**

<details>
<summary>点击展开</summary>

**逐项命中判定**：

| 扣分项 | 条件命中？ | 得分 |
|---|---|---|
| 未认领（25分） | claimed = false ✅ | 25 |
| 无网站（20分） | website 为空 ✅ | 20 |
| 低于同行评分 0.3+（15分） | 4.2 − 3.8 = 0.4 ≥ 0.3 ✅ | 15 |
| 差评率 ≥15%（15分） | 18% ≥ 15% ✅ | 15 |
| 评论数 < 同行中位 40%（12分） | 12 / 58 ≈ 20.7% < 40% ✅ | 12 |
| 照片 ≤3（8分） | photo = 5 ❌ | 0 |
| 无营业时间（10分） | 有 hours ❌ | 0 |
| 单一品类（5分） | 3 个品类 ❌ | 0 |
| 无服务项（5分） | 有 service_options ❌ | 0 |

**总分** = 25+20+15+15+12 = **87 分**

**判定**：87 ≥ 30，进入候选名单。这是一个典型的"高意向客户"——门店几乎没有任何线上维护痕迹。

**最容易忽略的隐藏扣分项**：**"低于同行评分 0.3+"（15分）**。很多开发者只关注显式的布尔字段（claimed / website），忽略了对比型指标。但它实际反映了该商家在同行中处于明显劣势——这类商家对"线上优化服务"的付费意愿往往最高。

</details>

---

## Code 逻辑改错题（2 道）

### 第 4 题 · 轮询状态机 bug

**题干**：以下是模板中"Bright Data 进度轮询"节点的简化代码，存在 **2 处逻辑错误**，请指出并给出修复代码。

```js
// 环境变量：BRIGHT_DATA_API_KEY
const MAX_POLLS = 20;
const POLL_INTERVAL_MS = 20000;

let snapshotId = $input.first().json.snapshot_id;
let status = 'running';

for (let i = 0; i < MAX_POLLS && status !== 'ready'; i++) {
  const response = await $http.request({
    method: 'GET',
    url: `https://api.brightdata.com/datasets/v3/progress/${snapshotId}`,
    headers: { Authorization: `Bearer ${$env.BRIGHT_DATA_API_KEY}` },
  });

  // 错误点 1：只检查 HTTP 状态码
  if (response.status !== 200) {
    throw new Error(`轮询失败: HTTP ${response.status}`);
  }

  status = response.body.status; // 假设为 running / ready / failed

  if (status === 'failed') {
    throw new Error('Bright Data 抓取失败');
  }

  if (status !== 'ready') {
    await new Promise(resolve => setTimeout(resolve, POLL_INTERVAL_MS));
  }
}

// 错误点 2：循环结束后直接返回
return [{ json: { snapshot_id: snapshotId, status } }];
```

**参考答案**

<details>
<summary>点击展开</summary>

**错误 1 —— 只检查 HTTP 状态码，未检查 body 内嵌 error**  
作者原话："Bright Data 常常 200 返回错误"。当前代码在 HTTP 200 时直接读取 `response.body.status`，若 body 是 `{ error: "Invalid snapshot" }`，`status` 为 undefined，循环会空转到超时。  
**修复**：增加 body 内 error 判断。

**错误 2 —— 循环超时后未显式处理，直接静默返回**  
当 20 次轮询全部跑完且 status 仍为 running，代码不会抛错，而是把 `status: 'running'` 返回下游——下游会拿不完整的数据继续执行。  
**修复**：循环结束后若 status 不是 ready，应主动抛异常。

**修复后完整代码**：

```js
const MAX_POLLS = 20;
const POLL_INTERVAL_MS = 20000;

let snapshotId = $input.first().json.snapshot_id;
let status = 'running';

for (let i = 0; i < MAX_POLLS; i++) {
  const response = await $http.request({
    method: 'GET',
    url: `https://api.brightdata.com/datasets/v3/progress/${snapshotId}`,
    headers: { Authorization: `Bearer ${$env.BRIGHT_DATA_API_KEY}` },
  });

  // 修复点 1：即使 HTTP 200，也要检查 body 内错误
  if (response.status !== 200 || response.body.error) {
    throw new Error(
      `轮询失败: HTTP ${response.status}, body error: ${response.body.error || 'none'}`
    );
  }

  status = response.body.status;

  if