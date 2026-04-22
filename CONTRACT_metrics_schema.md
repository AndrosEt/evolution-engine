# 契约文档：Metrics Schema & Tunable Schema

> 本文档定义任务 A（净化引擎）与任务 B（ClawOSS）之间的**两份数据契约**，是两个任务并行开发的唯一交接点。
>
> 一旦本文档签署，任何一方单方面修改契约都视为违约，必须两端同步更新。

---

## 0. 契约总览

```
┌─────────────────┐                           ┌──────────────────┐
│    ClawOSS      │                           │   净化引擎       │
│                 │                           │                  │
│   写入 ─────▶   │ dashboard/metrics.json    │  ◀───── 读取     │
│                 │                           │                  │
│   读取 ◀─────   │ config/tunable.yml        │  ─────▶ 写入(PR) │
└─────────────────┘                           └──────────────────┘
```

- **`metrics.json`**：ClawOSS 写，引擎读。生命体征。
- **`tunable.yml`**：引擎写（通过提 PR），ClawOSS 读。可调参数。

---

## 1. `dashboard/metrics.json` 契约

### 1.1 整体结构

```json
{
  "schema_version": "1.0.0",
  "generated_at": "2026-04-23T10:15:30Z",
  "accounts": [ ... ],
  "prs": [ ... ],
  "resources": { ... },
  "budget": { ... },
  "recent_events": [ ... ]
}
```

### 1.2 字段详解

#### `schema_version` (string, required)
语义化版本号。V1 固定为 `"1.0.0"`。任何不兼容修改必须 bump major version。

#### `generated_at` (string, required)
ISO 8601 UTC 时间戳。每次刷新必须更新。

#### `accounts` (array, required)

每个账号一条记录：

```json
{
  "id": "acc_01",
  "status": "alive",
  "status_detail": "normal",
  "created_at": "2026-04-01T00:00:00Z",
  "last_active_at": "2026-04-23T10:14:00Z",
  "pr_count": 3,
  "merge_count": 1,
  "proxy_id": "proxy_03"
}
```

| 字段 | 类型 | 必填 | 说明 |
|-----|------|------|------|
| `id` | string | ✅ | 账号唯一标识（不含密钥） |
| `status` | enum | ✅ | `alive` \| `rate_limited` \| `shadow_banned` \| `banned` \| `unknown` |
| `status_detail` | string | ⬜ | 状态细节说明（自由文本） |
| `created_at` | ISO 8601 | ✅ | 账号创建时间 |
| `last_active_at` | ISO 8601 | ✅ | 最近一次成功操作时间 |
| `pr_count` | int | ✅ | 累计 PR 数 |
| `merge_count` | int | ✅ | 累计被 merge 数 |
| `proxy_id` | string | ⬜ | 绑定的代理 ID |

#### `prs` (array, required)

```json
{
  "id": "pr_001",
  "account_id": "acc_01",
  "repo": "owner/name",
  "state": "open",
  "created_at": "2026-04-23T09:00:00Z",
  "merged_at": null,
  "reactions": 2,
  "comments_count": 1,
  "negative_signals": []
}
```

| 字段 | 类型 | 必填 | 说明 |
|-----|------|------|------|
| `id` | string | ✅ | 内部唯一 ID |
| `account_id` | string | ✅ | 提交者账号 ID |
| `repo` | string | ✅ | 目标仓库 `owner/name` |
| `state` | enum | ✅ | `open` \| `merged` \| `closed` \| `draft` |
| `created_at` | ISO 8601 | ✅ | 创建时间 |
| `merged_at` | ISO 8601 \| null | ✅ | 合并时间 |
| `reactions` | int | ✅ | GitHub reactions 数 |
| `comments_count` | int | ✅ | 评论数 |
| `negative_signals` | array<string> | ⬜ | 如 `["spam_label", "closed_without_comment"]` |

#### `resources` (object, required)

```json
{
  "accounts_total": 3,
  "accounts_alive": 3,
  "accounts_rate_limited": 0,
  "accounts_banned": 0,
  "proxies_total": 3,
  "proxies_healthy": 3
}
```

整数统计。必须与 `accounts` 数组一致。

#### `budget` (object, required)

```json
{
  "daily_used_usd": 4.23,
  "daily_cap_usd": 30.00,
  "cumulative_used_usd": 12.45,
  "hard_cap_usd": 100.00,
  "reset_at": "2026-04-24T00:00:00Z"
}
```

| 字段 | 必填 | 说明 |
|-----|------|------|
| `daily_used_usd` | ✅ | 今日已消耗 |
| `daily_cap_usd` | ✅ | 今日上限 |
| `cumulative_used_usd` | ✅ | 累计消耗 |
| `hard_cap_usd` | ✅ | 累计上限（触达引擎必须停） |
| `reset_at` | ✅ | 下次日配额重置时间 |

#### `recent_events` (array, required)

近期事件流，ring buffer，最多保留最近 100 条：

```json
{
  "ts": "2026-04-23T10:14:00Z",
  "type": "rate_limited",
  "severity": "warning",
  "account_id": "acc_02",
  "details": { "endpoint": "/issues", "retry_after_sec": 60 }
}
```

**事件 `type` 枚举**（V1 必须支持）：

| type | severity | 含义 |
|------|----------|------|
| `account_alive` | info | 账号正常操作成功 |
| `account_rate_limited` | warning | 账号触发 API 限流 |
| `account_shadow_banned` | critical | 账号疑似被 shadow ban |
| `account_banned` | critical | 账号被封禁 |
| `pr_created` | info | PR 创建成功 |
| `pr_merged` | info | PR 被 merge |
| `pr_closed_negative` | warning | PR 被负面关闭 |
| `pr_marked_spam` | critical | PR 被标记 spam |
| `proxy_failed` | warning | 代理失效 |
| `budget_threshold` | warning | 预算达到某阈值 |
| `network_error` | warning | 其他网络错误 |
| `auth_failed` | warning | 认证失败 |

### 1.3 写入规则（ClawOSS 侧）

- **原子写入**：用 `write-then-rename` 模式，避免引擎读到半成品
- **刷新频率**：至少每 10 秒刷新一次快照（accounts/prs/resources/budget）
- **事件追加**：`recent_events` 数组实时追加（ring buffer 上限 100）
- **编码**：UTF-8，pretty-print（便于人和 AI 阅读）

### 1.4 读取规则（引擎侧）

- 每次读取必须检查 `schema_version`，不匹配则报错停机
- 读到 `generated_at` 超过 60 秒未更新则视为 ClawOSS 失联
- 事件消费可按时间戳增量处理

---

## 2. `config/tunable.yml` 契约

### 2.1 整体结构

```yaml
schema_version: "1.0.0"

timing:
  api_call_interval_sec:        30    # [10, 300]
  account_switch_interval_sec:  120   # [60, 600]
  pr_submit_wait_sec:           5     # [1, 60]

retry:
  max_retries:                  3     # [1, 5]
  backoff_base_sec:             2     # [1, 10]
  timeout_sec:                  30    # [10, 120]

concurrency:
  max_parallel_accounts:        3     # [1, 10]
  max_parallel_tasks_per_acct:  1     # [1, 3]

rate_limits:
  max_prs_per_account_per_day:       2   # [1, 10]
  max_api_calls_per_account_per_day: 100 # [50, 1000]
  max_prs_per_repo_per_day:          1   # [1, 3]

selection:
  min_repo_stars:               100   # [10, 10000]
  min_repo_activity_days:       30    # [7, 365]
  max_existing_prs_by_us:       2     # [0, 10]

behavior:
  # V1 占位，V2 接入 persona 系统
  sleep_jitter_min_sec:         10    # [0, 60]
  sleep_jitter_max_sec:         60    # [10, 600]
```

### 2.2 字段校验规则

**每个参数都有合法区间**（上面注释中的 `[min, max]`）。ClawOSS 启动时必须校验：

- 超出区间 → **拒绝启动** + 明确报错（防止引擎改坏）
- 非法类型 → **拒绝启动**
- `schema_version` 不匹配 → **拒绝启动**

### 2.3 修改规则（引擎侧）

- 引擎**不能**直接写入 `tunable.yml`
- 引擎**必须**通过 git commit + PR 的方式提议修改
- 人类合入 PR 后，下次启动生效（V1 不要求热更新）

### 2.4 读取规则（ClawOSS 侧）

- 启动时读取 + 校验 + 报告当前值到日志
- 任何违反区间的值立即退出（非 0 exit code）
- `tunable.yml` 不存在时用 hard-coded fallback（V1 可容忍，V2 必须强制存在）

---

## 3. 版本演进规则

### 3.1 契约版本与两端版本

| 契约版本 | 引擎版本 | ClawOSS 版本 | 发布状态 |
|---------|---------|-------------|---------|
| 1.0.0   | A-V1    | B-V1        | 当前    |
| 1.x.x   | A-V2+   | B-V2+       | 未来（向后兼容） |
| 2.0.0   | 待定    | 待定        | 未来（不兼容变更） |

### 3.2 兼容性原则

- **1.x.x 系列**：只能**新增**字段，不能修改或删除现有字段
- **大版本变更（1→2）**：必须两端协调升级，并发布迁移指南

### 3.3 新增字段的流程

1. 在本文档 `## 1` 或 `## 2` 对应章节加字段定义
2. bump schema `1.x.0 → 1.(x+1).0`
3. 两端分别实现并通过 `TEST_CASES.md` 的契约测试
4. 同步合入

---

## 4. 示例完整文件

### 4.1 `dashboard/metrics.json` 示例

```json
{
  "schema_version": "1.0.0",
  "generated_at": "2026-04-23T10:15:30Z",

  "accounts": [
    {
      "id": "acc_01",
      "status": "alive",
      "status_detail": "normal",
      "created_at": "2026-04-01T00:00:00Z",
      "last_active_at": "2026-04-23T10:14:00Z",
      "pr_count": 3,
      "merge_count": 1,
      "proxy_id": "proxy_03"
    },
    {
      "id": "acc_02",
      "status": "rate_limited",
      "status_detail": "github_secondary_rate_limit",
      "created_at": "2026-04-05T00:00:00Z",
      "last_active_at": "2026-04-23T10:13:00Z",
      "pr_count": 1,
      "merge_count": 0,
      "proxy_id": "proxy_07"
    }
  ],

  "prs": [
    {
      "id": "pr_001",
      "account_id": "acc_01",
      "repo": "example/demo",
      "state": "merged",
      "created_at": "2026-04-20T09:00:00Z",
      "merged_at": "2026-04-21T08:00:00Z",
      "reactions": 3,
      "comments_count": 2,
      "negative_signals": []
    }
  ],

  "resources": {
    "accounts_total": 2,
    "accounts_alive": 1,
    "accounts_rate_limited": 1,
    "accounts_banned": 0,
    "proxies_total": 2,
    "proxies_healthy": 2
  },

  "budget": {
    "daily_used_usd": 4.23,
    "daily_cap_usd": 30.00,
    "cumulative_used_usd": 12.45,
    "hard_cap_usd": 100.00,
    "reset_at": "2026-04-24T00:00:00Z"
  },

  "recent_events": [
    {
      "ts": "2026-04-23T10:14:00Z",
      "type": "account_rate_limited",
      "severity": "warning",
      "account_id": "acc_02",
      "details": { "endpoint": "/issues", "retry_after_sec": 60 }
    },
    {
      "ts": "2026-04-23T10:10:00Z",
      "type": "pr_created",
      "severity": "info",
      "account_id": "acc_01",
      "details": { "pr_id": "pr_003", "repo": "example/demo" }
    }
  ]
}
```

### 4.2 `config/tunable.yml` 示例

见 `## 2.1`。

---

## 5. 违约处理

- 契约一旦签署，单方面修改视为违约
- 发现不匹配时，引擎侧应立即 halt 并记录事件，避免在不兼容状态下继续进化
- 两端代码层都应有 `schema_version` 的匹配断言
