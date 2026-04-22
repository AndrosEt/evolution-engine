# 测试用例：V1 验收标准

> 本文档是任务 A（净化引擎）与任务 B（ClawOSS 改造）的 V1 验收用例。所有用例都是"黑盒"测试，不关心内部实现，只关心行为与输出。
>
> **V1 所有测试用例必须 100% 通过**才能视为交付完成。

---

## 0. 测试分层

```
┌─────────────────────────────────────────────────────────┐
│ Layer 3: 联合验收测试（最关键，决定整体能用与否）      │
│   ├─ 整条闭环：ClawOSS 出 bug → 引擎发现 → 提 PR         │
├─────────────────────────────────────────────────────────┤
│ Layer 2: 集成测试（单任务端到端）                       │
│   ├─ 引擎：读 config → 提 PR                             │
│   └─ ClawOSS：启动 → 跑 30 min → 输出 metrics            │
├─────────────────────────────────────────────────────────┤
│ Layer 1: 单元测试（模块级别）                           │
│   ├─ 引擎：config_loader / observer / actor / judge / router │
│   └─ ClawOSS：metrics 写入器 / tunable 读取器 / status.sh │
└─────────────────────────────────────────────────────────┘
```

---

## 1. 任务 A（Evolution Engine）V1 测试

### A-V1-UT-001：配置加载器校验

**前置**：无

**步骤**：
1. 准备一份完整合法的 `evolution.yml`
2. 运行 `evolve validate --config evolution.yml`

**预期**：
- Exit code = 0
- 输出："Config valid"

**变体**：
- 删除 `mission` 字段 → 报错 "Missing required field: mission"
- `principles` 为空数组 → 报错 "principles must have at least 1 item"
- `hard_stops.budget_hard_cap_usd` = 0 → 报错 "must be > 0"
- `models.actor == models.judge` → 警告 "actor and judge should use different models"

---

### A-V1-UT-002：Observer 证据采集

**前置**：一个合法的 `evidence_sources` 列表，包含：
- 一个存在的 metrics.json（符合契约 schema）
- 一个可执行的 status.sh

**步骤**：
1. 调用 Observer 模块
2. 检查输出的上下文结构

**预期**：
- 上下文包含 metrics.json 的关键字段（accounts / prs / budget / recent_events）
- 上下文包含 status.sh 的输出
- 总 token 数在合理范围（< 5000 tokens）
- 损坏的 metrics.json（schema 不匹配）会被 Observer 报错并退出

---

### A-V1-UT-003：Actor 输出 patch

**前置**：模拟的 observation 上下文（含一个明显的 bug 信号，如"连续 3 次 rate_limited 事件"）

**步骤**：
1. 调用 Actor 模块
2. 解析返回的 patch

**预期**：
- 返回的 patch 是合法的 unified diff 格式 或 文件修改列表
- patch 里有针对 rate_limit 问题的代码修改（如增加 sleep 时间或减少并发度）
- 输出包含修改说明（rationale）

---

### A-V1-UT-004：Judge 独立性硬验证

**前置**：Actor 刚刚生成了一个 patch + reasoning

**步骤**：
1. 调用 Judge 模块
2. 检查传入 Judge 的 prompt 内容（通过日志 / hook 拦截）

**预期**：
- Judge 的输入中**不包含** Actor 的 chain-of-thought / reasoning 字段
- Judge 的输入中只有：①原始证据 ②最终 patch ③ principles ④ mission
- Judge 使用的模型 endpoint / API key 与 Actor **不同**（通过日志验证）
- Judge 的 system prompt 明确声明"独立审计"身份

**关键**：此用例是 **硬约束红线**，不通过则整个 V1 作废。

---

### A-V1-UT-005：Judge 输出 schema 校验

**前置**：Judge 已产出裁决

**步骤**：
1. 解析 Judge 的 JSON 输出
2. 按 `PRD_EvolutionEngine.md § 4.3` schema 校验

**预期**：所有必填字段齐全：
- `verdict ∈ {"PASS", "FAIL"}`
- `overall_score ∈ [0, 100]`
- `principle_scores` 为数组，长度等于 principles 数量
- 每条 principle_score 有 `score` 和 `reasoning`
- `confidence ∈ [0, 1]`
- `reasoning_summary` 非空

**变体**：Judge 返回非法 JSON → 引擎必须捕获并记录，不能崩溃

---

### A-V1-UT-006：Router 分流

**前置**：无

**步骤**：
1. 给 Router 一个 `verdict=PASS` 的裁决
2. 给 Router 一个 `verdict=FAIL` 的裁决

**预期**：
- PASS → 创建新 git 分支、commit patch、调 `gh pr create` 开 PR
- FAIL → 不提交任何代码，落盘历史记录，通知人类（stdout / webhook / 文件标记均可）

---

### A-V1-UT-007：Hard Stops 熔断

**前置**：无

**步骤**：连续触发以下场景：
1. 预算累计消耗达到 `hard_cap_usd`
2. 连续 5 次迭代 Judge 都 FAIL
3. 单日迭代次数达到 `max_iterations_per_day`

**预期**：
- 任一触发后，引擎立即停止（下一次 `evolve run` 启动时检查到 halt 状态也应拒绝继续）
- 输出清晰的停机原因
- 触发人工通知

---

### A-V1-IT-001：引擎端到端

**前置**：
- 一份完整的 `evolution.yml`
- 一个已埋 bug 的 ClawOSS 仓库快照
- metrics.json 反映了该 bug 引起的异常

**步骤**：
1. 运行 `evolve run --config evolution.yml`
2. 等待引擎执行完一次完整循环

**预期**：
- 引擎创建一个新分支（如 `evolution/fix-rate-limit-20260423`）
- 引擎 commit 了针对性的代码修改
- 引擎通过 `gh` 开了一个 PR
- PR 描述里包含 Judge 的 reasoning_summary
- 进化历史日志包含本轮完整记录

---

## 2. 任务 B（ClawOSS 改造）V1 测试

### B-V1-UT-001：PR#2 合入

**前置**：无

**步骤**：
1. 拉取主分支最新代码
2. 检查 PR#2 的变更是否全部在主分支

**预期**：
- PR#2 已 merged
- 主分支包含 PR#2 的 7 项改动（见 PR body）
- quickstart 文档可运行

---

### B-V1-UT-002：参数外部化

**前置**：ClawOSS 主分支

**步骤**：
```bash
rg -n 'time\.sleep\(\s*\d+\s*\)' --type py
rg -n 'retries\s*=\s*\d+' --type py
rg -n 'max_workers\s*=\s*\d+' --type py
# ... 对所有硬编码参数做类似检查
```

**预期**：
- 搜索结果为空（允许白名单，但必须在代码中有注释说明）
- 所有参数都从 `config/tunable.yml` 加载

---

### B-V1-UT-003：tunable.yml 校验

**前置**：`config/tunable.yml` 存在

**步骤**：
1. 正常启动 ClawOSS → 应成功
2. 把 `max_parallel_accounts` 改为 100（超出区间） → 启动
3. 把 `schema_version` 改为 `2.0.0` → 启动
4. 删除 `tunable.yml` → 启动

**预期**：
- 场景 1：启动成功，日志打印当前 tunable 值
- 场景 2：启动失败，报错 "max_parallel_accounts out of range [1, 10]"
- 场景 3：启动失败，报错 "schema_version mismatch"
- 场景 4：启动失败（V1 允许 fallback 到硬编码默认值，但必须告警）

---

### B-V1-UT-004：metrics.json 契约合规

**前置**：ClawOSS 运行中

**步骤**：
1. 读取 `dashboard/metrics.json`
2. 按 `CONTRACT_metrics_schema.md § 1` schema 校验

**预期**：
- schema_version = "1.0.0"
- 所有必填字段齐全
- `resources.accounts_total` 等于 `accounts` 数组长度
- `generated_at` 距离当前不超过 15 秒（证明持续刷新）

---

### B-V1-UT-005：status.sh

**前置**：ClawOSS 运行中（含 2~3 账号）

**步骤**：
```bash
bash scripts/status.sh
```

**预期**：
- Exit code = 0
- 输出包含账号池、代理池、预算三类健康度
- 执行时间 < 5 秒

---

### B-V1-UT-006：多账号并行崩溃隔离

**前置**：3 个账号并行，其中 1 个的 token 是故意错的

**步骤**：
1. 启动 ClawOSS
2. 观察运行 10 分钟

**预期**：
- 错 token 的账号在 metrics.json 里状态为 `auth_failed` 或 `unknown`
- 其他 2 个账号正常运行、创建 PR、刷新 metrics
- 整个进程不崩溃

---

### B-V1-IT-001：跑 30 分钟

**前置**：2~3 个真实账号 + 代理配置完成

**步骤**：
1. 启动 ClawOSS
2. 运行 30 分钟

**预期**：
- 进程全程不崩溃
- metrics.json 持续刷新（最长间隔不超过 15 秒）
- 至少 1 个 PR 被实际创建到目标仓库
- recent_events 中有至少 1 条 `pr_created` 事件
- 预算消耗 < daily_cap

---

## 3. 联合验收测试（V1 最终验收）

### UNION-V1-001：完整进化闭环（**最关键**）

**这是 V1 的最终验收场景。通过此用例即可认为 V1 交付成功。**

#### 前置条件

1. ClawOSS B-V1 已交付并部署（2~3 账号跑通）
2. 引擎 A-V1 已交付
3. 一份针对 ClawOSS 的 `evolution.yml`（见 `PRD_EvolutionEngine.md § 10`）

#### 步骤

**Step 1：埋 bug**

人为把 `config/tunable.yml` 中的 `api_call_interval_sec` 改为 `1`（极短，必触发 rate limit），然后重启 ClawOSS。

**Step 2：观察**

运行 ClawOSS 15 分钟。预期 metrics.json 中：
- 出现多条 `account_rate_limited` 事件
- 至少 1 个账号的 status 变为 `rate_limited`

**Step 3：触发引擎**

运行 `evolve run --config evolution.yml`。

**Step 4：验证引擎行为**

| 检查点 | 预期结果 |
|-------|---------|
| Observer 读到证据 | 进化日志中出现对 rate_limited 事件的引用 |
| Actor 生成 patch | patch 针对 `api_call_interval_sec` 做了增大（如从 1 改为 30+） |
| Judge 独立审计 | Judge 的日志/trace 与 Actor 分离；Judge 输出 JSON 合规 |
| Judge verdict | PASS（此场景修改合理） |
| Router 开 PR | gh 上出现一个新 PR，标题包含 "fix rate limit" 或类似语义 |
| PR 描述质量 | 包含修改原因、涉及文件、Judge 的 reasoning_summary |

**Step 5：人类合入**

人类 review 后合入 PR，重启 ClawOSS。

**Step 6：验证进化生效**

观察 15 分钟，预期：
- `account_rate_limited` 事件显著减少
- 账号状态恢复 `alive`

#### 判定

**全部 6 步通过 → V1 验收成功**

任何一步失败 → 必须定位到具体模块（引擎侧 or ClawOSS 侧）并修复后重跑。

---

### UNION-V1-002：硬兜底（熔断验证）

**前置**：完成 UNION-V1-001

**步骤**：
1. 在 ClawOSS 埋一个**根本无法通过修改 tunable 解决**的 bug（比如写一个 syntax error）
2. 让 Actor 提 patch，Judge 拒绝
3. 重复触发 5 次

**预期**：
- 第 5 次后，`hard_stops.max_consecutive_failures` 触发
- 引擎停机
- 输出明确的熔断原因
- 此后 `evolve run` 启动会检测到 halt 状态并拒绝执行（需人工 reset）

---

## 4. 验收 checklist（总览）

### 任务 A V1

- [ ] A-V1-UT-001 ~ A-V1-UT-007 全部通过
- [ ] A-V1-IT-001 通过
- [ ] 代码目录结构符合 `PRD_EvolutionEngine.md § 7`
- [ ] README 能让一个新人 30 分钟内跑起来

### 任务 B V1

- [ ] B-V1-UT-001 ~ B-V1-UT-006 全部通过
- [ ] B-V1-IT-001 通过
- [ ] 代码 grep 验证无硬编码参数
- [ ] `tunable.yml` schema 校验工作正常

### 联合

- [ ] UNION-V1-001 通过（整条闭环）
- [ ] UNION-V1-002 通过（熔断）
- [ ] 两份契约（metrics.json / tunable.yml）两端实现一致

---

## 5. V2 测试用例骨架（占位，V2 启动时再填充）

### A-V2 预留

- 事件触发正确性
- 多轮 Actor↔Judge 重试收敛性
- 自动 merge + 回滚的安全性
- 长期记忆的命中率

### B-V2 预留

- 20+ 账号并发稳定性
- Persona 行为多样性
- Target repo 选择质量
- 反检测有效性（账号存活率）

### UNION-V2 预留

- 长时间运行的进化累积效果（跑 1 周的进化历史评分曲线）
- 引擎无人值守持续运行 24h 的稳定性
