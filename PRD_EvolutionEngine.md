# PRD：即插即用的自我进化引擎（Evolution Engine）

> 本文档是任务 A 的需求规范。面向开发者与 AI 协作落地。
>
> 配套文档：
> - `PRD_ClawOSS_Evolvable.md`（任务 B：第一个被进化的业务靶子）
> - `CONTRACT_metrics_schema.md`（两端共享的数据契约）
> - `TEST_CASES.md`（验收测试）

---

## 0. 一句话定义

一个**与业务无关**的组件：读取一份 `evolution.yml` 配置文件，就能让任意代码仓库具备**持续自我净化（self-purification）**的能力——自动观测问题、修改代码、提交 PR、回滚止损、记录进化。

---

## 1. 背景与愿景

### 1.1 长期愿景

任何运行中的软件系统，只要挂载本组件并提供一份配置文件，就能获得持续"进化"的能力。进化的方向由用户通过**自然语言原则**声明，进化的量化方式由 AI 自己设计，人类只守住**方向**和**底线**。

### 1.2 核心哲学

| 交给人类的 | 交给 AI 的 |
|-----------|-----------|
| 方向（mission） | 如何理解方向 |
| 原则优先级（principles） | 如何把原则量化成分数 |
| 资源清单（resources） | 如何使用、轮换、节约资源 |
| 证据入口（evidence） | 从证据里看到什么、忽略什么 |
| 保命边界（hard_stops） | 在边界内如何优化 |
| 角色分工（models） | 如何写代码、如何审计代码 |

**只给方向和底线，能力交给 AI。**

### 1.3 系统流程

```
┌─────────────┐    ┌───────────┐    ┌──────────┐    ┌──────────┐
│  Observer   │───▶│   Actor   │───▶│  Judge   │───▶│  Router  │
│ (读证据)    │    │ (改代码)  │    │ (独立审计)│    │(PR/熔断) │
└──────┬──────┘    └───────────┘    └──────────┘    └─────┬────┘
       │                                                   │
       │            ┌──────────────────────┐               │
       └───────────▶│   Hard Stops 兜底     │◀──────────────┘
                    │  (预算/失败次数/时间) │
                    └──────────────────────┘
```

---

## 2. 版本范围

### 2.1 V1 范围（1 天内完成 + 必须能用）

**V1 目标**：让 ClawOSS 跑起来，并在其出现异常时**引擎能自动生成一个可合并的 PR**。

V1 **必须**有：
- CLI：`evolve run --config evolution.yml`
- 读配置 + 扫证据 → 构建上下文
- Actor（写代码的 Agent）：根据上下文生成 patch
- **Judge（独立模型/独立会话的审计 Agent）**：按 principles 优先级给 patch 结构化打分
- 路由：Judge `PASS` → 开 PR；Judge `FAIL` → 熔断、通知人类
- Hard stops：预算上限、最大迭代数、连续失败次数
- 手动触发（CLI / cron 即可）
- 本地进化历史日志（JSONL 落盘）
- 配置文件支持 6 个字段（见第 3 节）

V1 **不做**：
- 事件触发（先手动跑）
- 多轮 Actor↔Judge 重试
- 自动 merge + 灰度 + 自动回滚
- 长期记忆 / 向量数据库
- Web UI / Dashboard

### 2.2 V2 范围（V1 稳定后再做）

- 事件驱动：监听 evidence 变化自动触发
- Actor↔Judge 多轮重试（Judge 反馈给 Actor 改到过关）
- 自动 merge + 灰度发布 + 恶化自动回滚
- 进化评分历史、低分复盘机制
- 长期记忆（sqlite / vectorDB）
- 可视化 dashboard

### 2.3 V1 明确的"能用"标准

参见 `TEST_CASES.md` 的 V1 联合验收场景：
> 用 ClawOSS 跑 2~3 个账号 30 分钟 → 人为造一个会导致账号被限流的参数错误 → 引擎读 `metrics.json` 看到问题 → Actor 提 patch → Judge 审核通过 → 自动开一个 PR → 人类合入。

---

## 3. 配置文件规范（`evolution.yml`）

### 3.1 总原则

- **6 个字段**，不多一个也不少一个
- 每个字段都必须在 schema 校验层做**缺失检测**和**格式提示**
- 每个字段文档都按统一模板说明："作用 / 为什么要填 / 怎么填才算好 / 常见坑"

### 3.2 完整字段说明

#### 字段 1：`mission` —— 使命陈述

```yaml
mission: |
  运营一批 GitHub 账号，持续向真实的优质开源项目贡献有用的 PR。
  让账号长期存活并积累真实声誉，避免被平台风控识别。
```

| 项 | 说明 |
|---|---|
| 作用 | 告诉 AI 整个系统在为什么而努力。所有判断的总北极星。 |
| 为什么要填 | 没有 mission，Actor 和 Judge 都会盲目；不同 AI 调用之间语义漂移无法收敛。 |
| 怎么填才算好 | 1 段自然语言，3~5 句话；说清楚"做什么 + 为什么做 + 做给谁用"。 |
| 常见坑 | ❌ 只写"提升代码质量"这种空话；❌ 写成 TODO 列表；✅ 要有明确的业务语义和终极价值。 |

#### 字段 2：`principles` —— 原则（有序优先级）

```yaml
principles:
  - priority: 1
    rule: "账号绝不能被封禁或 Shadowban"
  - priority: 2
    rule: "行为必须拟人化，杜绝机械化特征"
  - priority: 3
    rule: "PR 必须对目标项目有实质价值，禁止水 PR"
  - priority: 4
    rule: "PR 被维护者真心合并或获得正面 review"
  - priority: 5
    rule: "Token 与基础设施成本与产出价值匹配"
```

| 项 | 说明 |
|---|---|
| 作用 | 告诉 Judge 按什么原则打分；冲突时按 priority 排序。 |
| 为什么要填 | 真实世界永远有冲突（比如"快出 PR"vs"账号安全"），没有优先级 AI 就会陷入摇摆。 |
| 怎么填才算好 | 3~7 条；每条一句话；priority 数字越小越重要；最重要的通常是"安全/不作恶"。 |
| 常见坑 | ❌ 写成具体的分数规则（"封号 -100"），这样就退化成硬规则了；✅ 只写方向和相对重要性，让 Judge 自己量化。 |

#### 字段 3：`resources` —— 资源池

```yaml
resources:
  accounts:
    path: "./workspace/resources/accounts.enc.json"
    health_check: "./scripts/check_account_health.sh"
    description: "GitHub 账号池，含用户名、token、绑定代理"

  proxies:
    path: "env:PROXY_PROVIDER_URL"
    health_check: "./scripts/check_proxy_latency.sh"
    description: "住宅 IP 代理池"

  target_repos:
    path: "./workspace/resources/target_repos.yml"
    description: "白名单开源项目列表"

  personas:
    path: "./workspace/resources/personas/"
    description: "每账号一份拟人画像"

  budget:
    daily_usd: 30
    hard_cap_usd: 100
```

| 项 | 说明 |
|---|---|
| 作用 | 告诉 AI 系统有哪些可消耗的资源、在哪找、怎么查健康度。 |
| 为什么要填 | 资源状态本身就是反馈信号（账号存活率下降时要触发进化）；资源约束决定进化策略的保守/激进程度。 |
| 怎么填才算好 | 对每类资源声明 `path`（去哪找）+ `health_check`（怎么知道健康）+ `description`（一句话描述）。`budget` 必须有 daily 和 hard_cap。 |
| 常见坑 | ❌ 把明文密钥写在 path 里；✅ 用 `env:VAR_NAME` 或加密文件路径。❌ 不写 health_check 导致 Judge 看不到资源状态；✅ 哪怕是 `echo ok` 的占位脚本也要有。 |

**权限边界（V1 硬规则）：**
- AI 可以**消耗**资源（必须记账）
- AI **不能**直接补充资源，只能提出**补充请求**（写到 issue / 通知人类）

#### 字段 4：`evidence_sources` —— 证据来源

```yaml
evidence_sources:
  - "./dashboard/metrics.json"
  - "./scripts/status.sh"
  - "github_api"
```

| 项 | 说明 |
|---|---|
| 作用 | 告诉 AI 去哪里"看世界"。Observer 会扫描并格式化这些来源喂给 Actor/Judge。 |
| 为什么要填 | 没有现实感，AI 会在幻觉里优化。证据源越丰富，进化决策越精准。 |
| 怎么填才算好 | 列出所有"能反映系统健康"的入口：日志文件、健康检查命令、外部 API。每条独立可读。 |
| 常见坑 | ❌ 塞一整个 `./logs/` 目录（token 爆炸）；✅ 指向结构化的汇总文件（如 metrics.json）。❌ 只给快照不给事件流；✅ 既要当前状态也要近期事件。 |

#### 字段 5：`hard_stops` —— 保命边界

```yaml
hard_stops:
  budget_hard_cap_usd: 100          # 累计消耗超此值立即停
  max_consecutive_failures: 5       # 连续 N 次 CI/Judge 拒绝就停
  max_iterations_per_day: 50        # 单日最多进化次数
  on_trigger: "halt_and_notify"     # 触发后的行为
```

| 项 | 说明 |
|---|---|
| 作用 | 防止 AI 在奇怪的局部最优里烧钱 / 死循环 / 造成不可逆破坏。 |
| 为什么要填 | 这是唯一**不能交给 AI**的部分——因为 AI 自己决定什么时候停就像让刹车踏板自己决定什么时候踩。 |
| 怎么填才算好 | 至少覆盖"钱、次数、时间、连续失败"4 个维度。宁可严格，不可宽松。 |
| 常见坑 | ❌ 只设 budget 不设 max_consecutive_failures，导致死循环烧钱；✅ 多维度冗余兜底。 |

#### 字段 6：`models` & `safety_mode` —— 角色分工

```yaml
models:
  actor: "claude-sonnet-4"
  judge: "claude-opus-4"   # 必须与 actor 不同（至少是独立会话）

safety_mode: "human_in_the_loop"  # V1 强制此模式
```

| 项 | 说明 |
|---|---|
| 作用 | 声明 Actor 和 Judge 用什么模型；声明是否需要人类合闸。 |
| 为什么要填 | Actor/Judge **必须**分离（见第 4 节硬约束）。V1 阶段必须 human-in-the-loop，不给自动 merge。 |
| 怎么填才算好 | Judge 最好用比 Actor 更强的模型（Opus > Sonnet）；V1 阶段 safety_mode 固定为 `human_in_the_loop`。 |
| 常见坑 | ❌ 为了省钱 actor/judge 用同一个模型同一个会话（违反硬约束）；❌ V1 就激进地用 auto_merge。 |

---

## 4. Actor / Judge 分离 —— 硬约束（不可妥协）

### 4.1 为什么必须分离

既当运动员又当裁判会导致 AI 为自己的 patch 辩护，验收失效。这是系统安全的底线。

### 4.2 具体约束

| 约束 | 实现要求 |
|------|---------|
| **模型独立** | Actor 和 Judge 必须使用不同模型；或至少不同 API key / 不同会话 ID 的同模型调用。 |
| **上下文隔离** | Judge **不能**看到 Actor 的推理链（chain of thought）。Judge 只接收：①原始证据 ②最终 patch ③ principles。 |
| **Prompt 互斥** | Judge 的 system prompt 必须强调"独立审计者"身份，明确禁止"体谅 Actor 的难处"。 |
| **结构化输出** | Judge 必须输出固定 JSON schema（见 4.3）。 |
| **可追溯** | Actor 的提议与 Judge 的裁决都落盘到进化历史，保留给后续分析。 |

### 4.3 Judge 输出 schema

```json
{
  "verdict": "PASS | FAIL",
  "overall_score": 0,
  "principle_scores": [
    { "priority": 1, "rule": "...", "score": 0, "reasoning": "..." }
  ],
  "top_risks": ["..."],
  "confidence": 0.0,
  "reasoning_summary": "..."
}
```

- `verdict`: 是否允许进入 Router 开 PR
- `principle_scores`: 对每条 principle 的独立评分与理由
- `top_risks`: 即便 PASS，也要列出风险点
- `confidence`: Judge 对自己判断的信心（低于阈值触发人工复核，V1 阈值 0.5）

---

## 5. V1 工程拆解（模块维度）

建议按以下模块拆分实现，每个模块可独立测试：

| 模块 | 职责 | 预估工作量 |
|-----|------|-----------|
| `config_loader` | 读 `evolution.yml` + schema 校验 + 默认值填充 | 30 min |
| `observer` | 扫 evidence_sources、拉 metrics、跑 status 脚本、汇总成上下文 | 1 h |
| `actor` | 调 Actor 模型，带 mission + principles + 证据，输出 patch | 1 h |
| `judge` | 调 Judge 模型（独立会话），按 schema 输出裁决 | 1 h |
| `router` | 根据 verdict 分流：PASS→建分支/commit/开 PR；FAIL→熔断+通知 | 1 h |
| `hard_stops` | 单例状态机：预算/失败次数/时间检查，任一触发则全局停 | 30 min |
| `history` | JSONL 落盘每轮的 observation / patch / verdict / outcome | 30 min |
| `cli` | `evolve run` 入口，串起所有模块 | 30 min |

**合计 ~6 小时**，Opus 4.6/4.7 协作可在 1 天内完成。

## 6. 技术选型建议（非强制）

- **语言**：Python（生态与 git/gh 集成成熟）或 TypeScript
- **LLM 调用**：官方 SDK（Anthropic / OpenAI）
- **Git 操作**：直接调用 `gh` CLI（最简单）
- **配置**：`pyyaml` + `pydantic` 做 schema 校验
- **落盘**：JSONL 进历史文件 + stdout 打印

## 7. 目录结构建议

```
evolution-engine/
├── src/
│   ├── cli.py
│   ├── config_loader.py
│   ├── observer.py
│   ├── actor.py
│   ├── judge.py
│   ├── router.py
│   ├── hard_stops.py
│   └── history.py
├── prompts/
│   ├── actor_system.md
│   └── judge_system.md
├── tests/
│   └── test_v1.py
├── examples/
│   └── evolution.yml        # 面向 ClawOSS 场景的示例
└── README.md
```

---

## 8. V1 验收标准（硬性）

必须全部满足（详见 `TEST_CASES.md`）：

- [ ] CLI `evolve run --config evolution.yml` 可以成功启动
- [ ] 配置缺字段时给出清晰的错误提示
- [ ] Actor 和 Judge 使用不同模型（或不同会话），日志可证明隔离
- [ ] 能从 `metrics.json` 读取证据并在输出中引用
- [ ] 人为在 ClawOSS 造一个能导致账号被限流的 bug，引擎能提出并合理 PR
- [ ] Judge 给出结构化的 JSON verdict（schema 完整）
- [ ] `hard_stops` 任一条件触发后引擎立即停止并通知
- [ ] 进化历史日志可读、可追溯

---

## 9. 非目标（V1 明确不做）

以下内容在 V1 明确**不实现**，避免延期：

- ❌ 自动 merge 到主分支
- ❌ 灰度发布
- ❌ 自动回滚
- ❌ 事件触发（V1 只支持手动/cron）
- ❌ 多轮 Actor↔Judge 对话重试
- ❌ 长期记忆 / 向量数据库
- ❌ 可视化 UI
- ❌ 分布式 / 多实例协同

所有上述内容挪到 V2。

---

## 10. 附录：V1 示例 `evolution.yml`（挂载到 ClawOSS）

```yaml
mission: |
  运营一批 GitHub 账号，持续向真实的优质开源项目贡献有用的 PR。
  账号长期存活并积累真实声誉，避免被平台风控识别。

principles:
  - priority: 1
    rule: "账号绝不能被封禁或 Shadowban"
  - priority: 2
    rule: "行为必须拟人化，杜绝机械化特征"
  - priority: 3
    rule: "PR 必须对目标项目有实质价值，禁止水 PR"
  - priority: 4
    rule: "PR 被维护者真心合并或获得正面 review"
  - priority: 5
    rule: "Token 与基础设施成本与产出价值匹配"

resources:
  accounts:
    path: "./workspace/resources/accounts.enc.json"
    health_check: "./scripts/check_account_health.sh"
  proxies:
    path: "env:PROXY_PROVIDER_URL"
    health_check: "./scripts/check_proxy_latency.sh"
  budget:
    daily_usd: 30
    hard_cap_usd: 100

evidence_sources:
  - "./dashboard/metrics.json"
  - "./scripts/status.sh"

hard_stops:
  budget_hard_cap_usd: 100
  max_consecutive_failures: 5
  max_iterations_per_day: 20
  on_trigger: "halt_and_notify"

models:
  actor: "claude-sonnet-4"
  judge: "claude-opus-4"

safety_mode: "human_in_the_loop"
```
