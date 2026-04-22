# PRD：ClawOSS 可进化改造（Evolvable ClawOSS）

> 本文档是任务 B 的需求规范。目标是把 ClawOSS 从"单次跑一个 Agent"改造成"可被净化引擎持续进化的 GitHub 高质量 PR 孵化器"。
>
> 配套文档：
> - `PRD_EvolutionEngine.md`（任务 A：净化引擎）
> - `CONTRACT_metrics_schema.md`（两端共享的数据契约）
> - `TEST_CASES.md`（验收测试）
>
> 源仓库：https://github.com/billion-token-one-task/ClawOSS
> 前置 PR：https://github.com/billion-token-one-task/ClawOSS/pull/2（V1 要先合入）

---

## 0. 一句话定义

把 ClawOSS 改造成一个**可被进化引擎持续修改**的 GitHub PR 孵化系统：它负责**执行（养号 + 提 PR）**，并把自己的生命体征暴露给引擎；引擎负责**持续修改它自己**。

---

## 1. 背景与定位

### 1.1 ClawOSS 现状

- 具备单 Agent 养号 + 提 PR 的基础能力
- 具备 dashboard + 指标上报框架（PR#2）
- 具备 Token 预算追踪、模型路由

### 1.2 本次改造的定位

ClawOSS 是**第一个被净化引擎进化的业务靶子**。改造有两层意义：
1. **业务层**：让它能批量、持续、稳定地养号并孵化高质量 PR
2. **架构层**：让它的核心参数和行为**可被引擎修改**，让引擎能通过读它的生命体征发现问题

### 1.3 与引擎的接口

**两个契约**（定义见 `CONTRACT_metrics_schema.md`）：

- `dashboard/metrics.json`：ClawOSS 写，引擎读 —— 生命体征
- `config/tunable.yml`：引擎写，ClawOSS 读 —— 可调参数

```
┌─────────────────┐                      ┌──────────────────┐
│    ClawOSS      │──── metrics.json ───▶│   净化引擎       │
│ (养号 + 提 PR)  │                      │ (Actor + Judge)  │
│                 │◀─── tunable.yml ─────│                  │
└─────────────────┘                      └──────────────────┘
     执行者                                     进化者
```

---

## 2. 版本范围

### 2.1 V1 范围（1 天内完成 + 必须跑通）

**V1 目标**：ClawOSS 能用 2~3 个账号稳定跑 30 分钟，输出结构化生命体征，且其行为参数可被外部修改。

V1 **必须**交付：

1. **合入 PR#2**（修复 model routing bugs + per-model cost tracking + quickstart doc）
2. **参数外部化**：把硬编码的时序/重试/并发参数提到 `config/tunable.yml`，让引擎能修改
3. **生命体征输出**：按 `CONTRACT_metrics_schema.md` 的 schema 写 `dashboard/metrics.json`
4. **健康检查脚本**：`scripts/status.sh` 一行命令输出账号池/代理池/预算当前健康度
5. **多账号并行骨架**：支持同时跑 2~3 个账号（每个有独立 token + 代理），不崩溃
6. **跑通端到端**：在本地实际跑 30 分钟，生成至少 1 个 PR（可以是到任意测试仓库）

V1 **不做**：
- 50+ 账号规模
- Persona 系统
- 智能 target repo 发现
- 反检测进阶（指纹、UA 池）
- 账号自动注册
- 高质量 PR 生成（V1 生成的 PR 可以质量一般，但流程要跑通）

### 2.2 V2 范围（V1 稳定后再做）

- **账号规模**：扩展到 20+ 并行
- **Persona 系统**：每账号绑定独立拟人画像（作息、兴趣、代码风格）
- **Target Repo 智能发现**：按语言/活跃度/friendly label 自动筛选贡献对象
- **反检测进阶**：时序抖动、UA 池、浏览器指纹多样化、IP-账号绑定策略
- **账号自动注册**：Agent 可自主注册新账号（或至少发起请求）
- **高质量 PR 生成**：真正读代码、理解项目、写有价值修改

---

## 3. 可进化性原则（V1 的核心架构约束）

### 3.1 什么叫"可进化"

代码中所有**将来可能需要被进化引擎调整**的参数，**必须**从代码中抽出来，放到 `config/tunable.yml`。

### 3.2 必须可调的参数（V1 清单）

按类别列出。工程师在改造时，把代码中所有硬编码的以下类型的值都抽出来：

| 类别 | 示例参数 |
|-----|---------|
| **时序** | API 调用间隔、账号之间的切换间隔、PR 提交后等待时间 |
| **重试** | 失败重试次数、重试退避时间、超时时间 |
| **并发** | 最大并行账号数、单账号最大并行任务数 |
| **风控规避** | 单账号每日最大 PR 数、单仓库最大 PR 数、单日最大 API 调用数 |
| **筛选阈值** | 目标仓库最低 star 数、最低活跃度、最大已有 PR 数 |
| **行为分布** | 随机休眠区间的上下限（为 V2 persona 预留） |

> **硬规则**：V1 代码审查时，若发现任何上述类别的参数仍硬编码在 `.py / .ts / .sh` 里，**必须重构**。

### 3.3 参数变更的生效机制

- `tunable.yml` 被修改后，下一次任务启动时必须重新读取（不要求热更新）
- 引擎通过 git commit 修改 `tunable.yml` 并提 PR；人类合入后下次启动生效
- `tunable.yml` 有 schema 校验，非法值拒绝启动（防止引擎改坏）

### 3.4 `tunable.yml` 示例

详见 `CONTRACT_metrics_schema.md`。此处给一个简化预览：

```yaml
timing:
  api_call_interval_sec: 30       # [10, 300]
  account_switch_interval_sec: 120 # [60, 600]
  pr_submit_wait_sec: 5

retry:
  max_retries: 3                  # [1, 5]
  backoff_base_sec: 2             # [1, 10]

concurrency:
  max_parallel_accounts: 3        # [1, 10]

rate_limits:
  max_prs_per_account_per_day: 2
  max_api_calls_per_account_per_day: 100
```

---

## 4. V1 工程拆解

按模块拆解，每个模块可独立验收：

### 4.1 合入 PR#2（预估 30 min）

- 审查现有 PR#2：https://github.com/billion-token-one-task/ClawOSS/pull/2
- 解决冲突（如有）
- 合入主分支
- 验证 quickstart 文档能跑通

### 4.2 参数外部化（预估 2 h）

- 扫描代码库找出所有硬编码的"时序/重试/并发/风控"参数
- 建立 `config/tunable.yml` + schema
- 提取一个统一的 `get_tunable(key)` 读取函数，替换代码中的硬编码值
- 增加非法值校验（启动时失败则明确报错）

### 4.3 Metrics 输出（预估 2 h）

- 按 `CONTRACT_metrics_schema.md` 定义的 schema 实现 metrics 写入器
- 在以下关键埋点位置触发写入：
  - 账号状态变化（alive / rate_limited / banned）
  - PR 状态变化（created / merged / closed）
  - 资源事件（代理失效、预算进度）
  - 错误事件（rate_limited, auth_failed, network_error）
- 写入策略：增量事件追加 + 全量快照覆盖（每 N 秒一次）

### 4.4 Status 脚本（预估 30 min）

- `scripts/status.sh` 输出（JSON 或 text 均可，但建议 JSON）：
  - 账号池：总数 / 存活 / 限流 / 封禁
  - 代理池：总数 / 健康
  - 预算：今日已用 / 今日上限 / 累计上限剩余
  - 最近错误事件数量

### 4.5 多账号并行骨架（预估 2 h）

- 支持 `accounts.enc.json` 配置多个账号
- 每账号绑定一个 proxy
- 并发度由 `tunable.yml.concurrency.max_parallel_accounts` 控制
- 账号间完全隔离（崩一个不影响其他）
- 打通端到端：能用 2~3 个账号各自提一个 PR

### 4.6 跑通 30 分钟（预估 1 h）

- 在本地实际运行 30 分钟
- 检查 metrics.json 持续更新
- 检查 status.sh 能反映真实状态
- 至少有 1 个 PR 被实际创建（可到测试仓库）

**合计 ~8 小时**，Opus 协作可在 1 天内完成。

---

## 5. V1 验收标准

参见 `TEST_CASES.md` B-V1 段。核心硬性指标：

- [ ] PR#2 已合入主分支
- [ ] 代码 grep 不再出现硬编码的时序/重试/并发参数
- [ ] `config/tunable.yml` 存在且有 schema 校验
- [ ] `dashboard/metrics.json` 符合契约 schema 且持续更新
- [ ] `scripts/status.sh` 能正确输出当前健康度
- [ ] 2~3 个账号并行跑 30 分钟不崩
- [ ] 至少创建 1 个真实 PR
- [ ] 引擎（A-V1）能读 metrics.json 并识别出 ClawOSS 的状态

---

## 6. 非目标（V1 明确不做）

- ❌ 20+ 账号规模
- ❌ Persona 系统
- ❌ 智能 target repo 发现
- ❌ 反检测进阶
- ❌ 账号自动注册
- ❌ 高质量 PR 生成（V1 PR 质量一般即可）
- ❌ 引擎的自动 merge（由引擎 PRD 保证）

---

## 7. V1→V2 过渡原则

V1 写代码时必须为 V2 留好扩展点：

- **Persona 系统的预埋**：在账号数据结构里预留 `persona_id` 字段，即使 V1 全部为 null
- **Target Repo 筛选**：V1 可以用静态白名单，但筛选函数接口要抽象，便于 V2 替换为智能筛选
- **反检测**：V1 的时序参数要支持"区间 + 随机"，即使 V1 取固定值，V2 切随机不改接口

这些预埋**不要求 V1 实现功能**，只要求**接口留好**。

---

## 8. 目录结构建议（改造后）

```
ClawOSS/
├── config/
│   ├── openclaw.json
│   └── tunable.yml            # ← 新增：引擎可修改的参数
├── workspace/
│   └── resources/
│       ├── accounts.enc.json  # ← 新增：账号池
│       └── target_repos.yml   # ← 新增：目标仓库白名单
├── dashboard/
│   └── metrics.json           # ← 按契约 schema 输出
├── scripts/
│   ├── status.sh              # ← 新增：一行健康度
│   ├── check_account_health.sh
│   └── check_proxy_latency.sh
└── (原有其他文件)
```
