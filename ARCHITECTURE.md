# ACM Tracker 项目架构文档

## 1. 项目概述

**ACM Tracker** 是一个面向 ACM 竞赛选手的多 OJ 刷题数据统一管理工具，支持 **Codeforces、洛谷 (Luogu)、AtCoder、牛客 (NowCoder)** 四大 OJ 平台。核心功能包括：自动同步提交记录、多维度能力评估、以及基于 LLM 的 AI 个性化学习规划。提供 **CLI 命令行** 和 **VSCode 扩展** 两种交互方式。

---

## 2. 四层架构总览

项目采用经典的分层架构，自顶向下分为四层：

```
用户界面层 ─── CLI (Typer + Rich)  /  VSCode Extension (Webview)
     │
核心业务层 ─── Config / Models(ORM) / Evaluator / AIPlanner / Database
     │
数据适配层 ─── AdapterRegistry → [Codeforces|Luogu|AtCoder|NowCoder]
     │
数据存储层 ─── SQLite (WAL) / config.yaml / 外部 API (OpenAI + OJ API)
```

### 2.1 各层职责

| 层次 | 职责 | 核心组件 |
|------|------|----------|
| 用户界面层 | 接收用户输入，展示结果 | `cli.py`（7 个命令）、VSCode 扩展 |
| 核心业务层 | 业务逻辑、数据模型、评估、规划 | `Config`、`Database`、`AbilityEvaluator`、`AIPlanner` |
| 数据适配层 | 统一不同 OJ 的数据格式 | `OJAdapter` 抽象基类 + 4 个实现 + `AdapterRegistry` |
| 数据存储层 | 持久化存储与外部 API 调用 | SQLite 数据库、YAML 配置文件、OpenAI API |

---

## 3. 模块详细拆解

### 3.1 用户界面层

**文件：** `cli.py`（507 行）

基于 **Typer** 构建的 CLI 入口，共 7 个子命令：

| 命令 | 功能 | 核心调用链 |
|------|------|-----------|
| `acm add <oj> <handle>` | 添加 OJ 账号并同步 | Config → AdapterRegistry → Database |
| `acm sync [oj]` | 同步提交记录 | AdapterRegistry → Database → 自动 AC 检测 |
| `acm profile` | 数据概况 | Database.get_user_profile() |
| `acm eval` | 能力评估报告 | AbilityEvaluator.evaluate() |
| `acm plan --target` | AI 生成学习计划 | AIPlanner.generate_plan() |
| `acm set-llm` | 配置 LLM API Key | Config.set_llm_key() |
| `acm config` | 管理配置 | Config 读写 |

**VSCode 扩展：** `vscode/package.json` 定义了 6 个命令（`showDashboard`、`sync`、`eval`、`genPlan`、`setHandle`、`showPlan`），通过 `Ctrl+Shift+A` 快捷键激活 Dashboard Webview 面板。

---

### 3.2 核心业务层

#### 3.2.1 Config (`core/config.py`)

配置管理器，加载优先级：

1. **环境变量**（`ACM_TRACKER_LLM_API_KEY` 等）
2. **`~/.acm-tracker/config.yaml`**（用户级）
3. **`config.yaml`**（项目级默认）

```python
Config
├── oj_mappings     → {codeforces, luogu, atcoder, nowcoder}
├── llm_config      → {provider, api_base, api_key, model}
├── storage         → {db_path, cache_dir}
├── sync            → {auto, interval_hours, cache_hours}
└── plan            → {default_daily_problems, weekly_contest, target}
```

#### 3.2.2 数据模型 (`core/models.py`)

采用 **Pydantic 业务模型 + SQLAlchemy ORM** 双轨制：

**Pydantic 模型（业务层传输）：**

| 模型 | 用途 |
|------|------|
| `NormalizedSubmission` | 统一的提交记录（跨 OJ 标准化） |
| `NormalizedProblem` | 统一的题目信息 |
| `AbilityReport` | 能力评估报告 |
| `StudyPlan` | 学习计划 |
| `PlanProgress` | 计划进度追踪 |

**SQLAlchemy ORM（持久化）：**

| ORM 类 | 表名 | 关键约束 |
|--------|------|----------|
| `SubmissionORM` | `submissions` | `uq_oj_submission`（唯一提交） |
| `ProblemCacheORM` | `problem_cache` | `uq_oj_problem_cache` |
| `PlanORM` | `plans` | `is_active` 标志 + JSON 进度字段 |

**核心枚举：**
- `OJSource`: codeforces / luogu / atcoder / nowcoder
- `Verdict`: AC / WA / TLE / MLE / RE / CE / PARTIAL / PENDING / UNKNOWN
- `DifficultyMap`: 800→入门, 1200→普及-, ... 2400→NOI+, 3000→神题

**Database 管理器：**
- 基于 SQLite（WAL 模式，支持并发读）
- 提供 Session 工厂、批量 upsert、计划进度追踪、AC 状态扫描等方法

#### 3.2.3 能力评估引擎 (`core/evaluator.py`)

`AbilityEvaluator` 从数据库提取提交记录，进行 6 个维度的分析：

| 维度 | 方法 | 说明 |
|------|------|------|
| 基础统计 | `evaluate()` | 总 AC 数、AC 率、最高难度 |
| 当前水平估算 | `_estimate_current_rating()` | 基于 AC 题目 rating 的分位数估算 |
| 主题掌握度 | `_calc_topic_mastery()` | 按 tag 统计 AC 率 × 0.7 + 尝试因子 × 0.3 |
| 难度梯度 | `_calc_difficulty_gradient()` | 按 200 分桶统计各段 AC 数量 |
| 薄弱点发现 | `_find_weak_points()` | 卡题（≥3 次未过）+ 低掌握度主题 |
| 月度趋势 | `_calc_monthly_trend()` | 按月统计去重 AC 题数 |

额外提供 `generate_radar_data()` 函数将主题掌握度归入 7 大类别（基础算法、数据结构、动态规划、图论、数学、字符串、搜索），适用于雷达图可视化（目前未被 CLI 调用，为 VSCode 扩展预留）。

#### 3.2.4 AI 规划引擎 (`core/planner.py`)

`AIPlanner` 是学习计划生成的核心：

**主流程（`generate_plan`）：**
1. 调用 `AbilityEvaluator.evaluate()` 获取用户能力画像
2. 获取已 AC 题目集合，经 `_filter_reusable_problems()` 智能去重
3. 构造 System Prompt + User Prompt（含用户画像、目标、可重做题目白名单）
4. 调用 OpenAI API（`response_format: json_object`）生成计划 JSON
5. **返回 DraftPlan 预览（不自动保存）**，等待用户确认
6. **`confirm_plan(draft_id)` 确认后**才写入 `plans` 表并分配 `daily_tasks`
7. LLM 调用失败时回退到 `_fallback_plan`

**预览确认流程（方案 B）：**

```
acm plan --target 1800
      │
      ▼
  AIPlanner.generate_plan()
      │
      ▼
  DraftPlan（临时，不写入 plans 表）
      │
      ├── CLI:  Rich 表格展示题单 → "确认导入？[Y/n/e 编辑]"
      │         ├── Y → confirm_plan() → 写入 plans + daily_tasks
      │         ├── n → 丢弃草稿
      │         └── e → 交互式编辑 → confirm_plan()
      │
      └── VSCode: Webview 预览面板 → 移除/调序 → 确认按钮
                └── acm plan confirm → 写入数据库
```

新增 CLI 子命令：

| 命令 | 功能 |
|------|------|
| `acm plan --target 1800` | 生成并展示预览，交互式确认 |
| `acm plan --target 1800 --force` | 跳过预览，直接导入（兼容旧行为） |
| `acm plan confirm` | 确认最近一次草稿 |
| `acm plan discard` | 丢弃草稿 |

**智能去重规则（`_filter_reusable_problems`）：**

问题：AI 可能推荐用户已经 AC 的题。简单排除所有已 AC 题会浪费高质量训练资源——有些题值得重做。

规则：已 AC 题按"可重做指数"过滤，只有同时满足以下条件的 AC 题才可被再次推荐：

| 条件 | 阈值 | 理由 |
|------|------|------|
| AC 正确率 < 阈值 | `< 70%`（即至少 30% 的提交非 AC） | 说明这道题用户不是稳定掌握 |
| 距上次 AC 时间 | `> 60 天` | 时间久远，记忆淡化，值得温习 |
| 提交总次数 | `≥ 3 次` | 至少挣扎过，确认是"有挑战的题"而非秒杀题 |

```python
def _filter_reusable_problems(submissions: list, now: datetime) -> list[str]:
    """
    返回可被重新推荐的问题 ID 白名单。
    未被列入白名单的 AC 题在 Prompt 中标记为"严禁推荐"。
    """
    reusable = []
    forbidden = []
    by_problem = groupby(submissions, key=lambda s: s.problem_id)
    for pid, subs in by_problem.items():
        ac_subs = [s for s in subs if s.verdict == Verdict.AC]
        if not ac_subs:
            continue  # 从未 AC 的题不受限制
        total = len(subs)
        ac_rate = len(ac_subs) / total
        last_ac = max(s.ctime for s in ac_subs)
        days_since_ac = (now - last_ac).days
        if ac_rate < 0.7 and days_since_ac > 60 and total >= 3:
            reusable.append(pid)
        else:
            forbidden.append(pid)
    return reusable, forbidden
```

**Prompt 注入逻辑：**
- `forbidden` 列表写入 System Prompt：「**严禁推荐以下题目**（用户已稳定掌握或近期刚 AC）：{forbidden_list}」
- `reusable` 列表写入 Prompt：「以下题目用户曾 AC 但掌握不牢，可酌情推荐 1~2 道用于温习：{reusable_list}」

**内置题库：** `_CLASSIC_PROBLEMS` 包含 CF 经典题目索引（800~2000 分共 10 个难度档位），每档 10 题，用于回退计划和题目推荐。

**精调机制（`refine_plan`）：** 支持基于用户反馈精调已有计划，将当前计划内容 + 完成状态 + 用户反馈 + 能力画像一起发送给 LLM 重新生成。

---

### 3.3 数据适配层

**设计模式：** 策略模式（抽象基类 + 具体实现 + 注册中心）

#### 3.3.1 基类 `OJAdapter` (`adapters/base.py`)

```python
class OJAdapter(ABC):
    oj: OJSource                    # OJ 标识
    RATE_LIMIT_INTERVAL = 0.0       # 速率限制间隔

    @abstractmethod
    def fetch_submissions(handle, limit) → List[NormalizedSubmission]
    @abstractmethod
    def fetch_problem(problem_id) → Optional[NormalizedProblem]

    def normalize_rating(raw_rating) → int   # 难度标准化为 CF rating
```

内置 `RateLimiter`（线程安全的速率控制）和 `_get_session()`（带指数退避重试的 HTTP Session）。

#### 3.3.2 四个适配器实现

| 适配器 | 数据来源 | 难度映射 | 特色机制 |
|--------|----------|----------|----------|
| `CodeforcesAdapter` | `codeforces.com/api` (REST) | 原生 CF rating | 严格 1s 限流，完整 Verdict 映射 |
| `LuoguAdapter` | `luogu.com.cn` (官方 API) | 洛谷难度 → CF rating | 基于 `luogu-api-docs` 项目，直接调用洛谷 REST API |
| `AtCoderAdapter` | `kenkoooo.com` (第三方 API) | `difficulty×0.8+800` | 内存缓存难度（5000 条/1h TTL） |
| `NowCoderAdapter` | `ac.nowcoder.com` (爬虫) | 牛客难度 → CF rating | UID 解析（支持用户名/数字ID） |

**关键设计决策：**
- 所有适配器将原始难度统一映射为 **CF rating 体系**（800~3500），保证跨 OJ 数据可比较
- Codeforces 使用官方 API，洛谷使用 `luogu-api-docs` 记录的官方 API，其余通过第三方 API/爬虫
- 每个适配器独立管理速率限制和 HTTP Session

#### 3.3.2.1 LuoguAdapter 参考实现（基于 luogu-api-docs）

**来源验证：** `0f-0b/luogu-api-docs`（GitHub ★99，194 commits，最后更新 2026-06-15）
- 项目活跃，社区认可度高（22 fork）
- 非官方但覆盖全面：题目、记录、用户、比赛、讨论、私信等 18 个 API 类别
- 文档与洛谷实际 API 行为保持同步（持续有人提 PR 更新）

**LuoguAdapter 关键 API 映射：**

| 功能 | luogu-api-docs 端点 | 当前 LuoguAdapter 实现 |
|------|---------------------|----------------------|
| 获取用户提交记录 | `GET /record/list?uid=X&page=Y` | **应使用此 API**，支持分页、按状态筛选 |
| 获取题目详情 | `GET /problem/:pid` | 用于 `fetch_problem()` |
| 获取用户信息 | `GET /user/:uid` | 用于验证用户名和获取 UID |
| 搜索用户 | `GET /api/user/search?keyword=X` | 替代当前的手动 UID 解析 |

**API 调用规范（来自 luogu-api-docs）：**
- 所有请求需设置 `User-Agent`（**不能包含 `python-requests`**，不能以 `Mozilla/` 开头）
- 响应类型为 `DataResponse` 的接口需参数 `_contentOnly` 或请求头 `x-luogu-type: content-only`
- 响应类型为 `LentilleDataResponse` 的接口需请求头 `x-lentille-request: content-only`
- POST 请求需 `Referer: https://www.luogu.com.cn/` + CSRF Token

**提交记录状态映射（luogu-api-docs → Verdict 枚举）：**

| 洛谷状态 | Verdict |
|----------|---------|
| 12 (Accepted) | AC |
| 14 (Wrong Answer) | WA |
| 15/16 (Time Limit Exceeded) | TLE |
| 17/18 (Memory Limit Exceeded) | MLE |
| 19/20 (Runtime Error) | RE |
| 21 (Compile Error) | CE |
| 其他 | UNKNOWN |

#### 3.3.3 注册中心 `AdapterRegistry` (`adapters/registry.py`)

```
AdapterRegistry
├── _adapters: Dict[OJSource, Type[OJAdapter]]   # 类注册
├── _instances: Dict[OJSource, OJAdapter]         # 实例缓存（单例）
├── get(oj)        → OJAdapter                     # 按枚举获取
├── get_by_name(name) → OJAdapter                  # 按字符串获取
└── all_adapters() → Dict                          # 获取全部
```

---

### 3.4 数据流全景

```
用户操作 (CLI/VSCode)
      │
      ▼
  cli.py 命令处理
      │
      ├── sync ──→ AdapterRegistry ──→ OJ API/爬虫
      │                  │
      │                  ▼
      │           NormalizedSubmission
      │                  │
      │                  ▼
      │           Database.batch_upsert → SQLite
      │
      ├── eval ──→ Database ──→ AbilityEvaluator ──→ AbilityReport
      │
      └── plan ──→ AbilityEvaluator + Database ──→ AIPlanner
                        │                              │
                        ▼                              ▼
               _filter_reusable_problems()       OpenAI API
                        │                              │
                        ▼                              ▼
                  智能去重列表                DraftPlan（草稿）
                        │                              │
                        ▼                              ▼
                  LLM Prompt              CLI 预览 / VSCode 预览
                                                   │
                                          ┌────────┼────────┐
                                          ▼        ▼        ▼
                                       确认(Y)   编辑(e)   丢弃(n)
                                          │        │
                                          ▼        ▼
                                  confirm_plan() ←┘
                                          │
                                          ▼
                                  plans 表 + daily_tasks 表
```

---

## 4. 技术栈总结

| 类别 | 技术选型 | 用途 |
|------|----------|------|
| CLI 框架 | Typer + Rich | 命令行交互、格式化输出 |
| 数据验证 | Pydantic v2 | 业务模型定义与校验 |
| ORM | SQLAlchemy 2.0 | 数据库映射与查询 |
| 数据库 | SQLite (WAL) | 本地持久化存储 |
| HTTP 客户端 | requests + httpx | OJ API 调用与爬虫 |
| AI 集成 | openai SDK | LLM 计划生成 |
| 配置管理 | PyYAML | YAML 配置文件读写 |
| VSCode 扩展 | TypeScript + VS Code API | IDE 集成面板 |
| 可视化 | matplotlib | 图表生成（需求预留） |

---

## 5. 每日打卡题单面板（2026-06-27 新增设计）

### 5.1 功能概述

在 VSCode 扩展中新增 **每日打卡题单 Webview 面板**，用于：
- 展示 AI 生成计划中当天需要完成的题目列表
- 每道题附带 **题目链接**（一键跳转 OJ 原题页面）
- 支持 **多状态打卡**（未开始 / 尝试中 / 已 AC / 已放弃）
- 7 天内可补卡
- 进度概览 + 本周热力图

### 5.2 新增数据模型

```python
# core/models.py 新增

class TaskStatus(str, Enum):
    NOT_STARTED  = "not_started"   # 未开始
    ATTEMPTING   = "attempting"    # 尝试中
    AC_COMPLETED = "ac_completed"  # 已 AC
    GAVE_UP      = "gave_up"       # 已放弃

class DailyTask(BaseModel):
    id: int
    plan_id: int
    problem_id: str              # OJ 题目 ID
    oj: OJSource
    problem_name: str
    difficulty: int              # CF rating 标准化
    difficulty_label: str        # 入门 / 普及- / ...
    problem_url: str             # 完整链接（由 OJAdapter 生成）
    tags: list[str]
    scheduled_date: date         # 计划完成日期
    status: TaskStatus
    notes: str = ""

class DailyPlanResponse(BaseModel):
    date: date
    total: int
    completed: int               # AC_COMPLETED 的数量
    tasks: list[DailyTask]
    progress_history: list[dict] # 近 7 天打卡统计
```

**SQLAlchemy ORM 新增表：**

| ORM 类 | 表名 | 关键约束 |
|--------|------|----------|
| `DailyTaskORM` | `daily_tasks` | `uq_plan_problem_date`（同一天同一计划不能重复分配同一题） |

### 5.3 核心业务层 — PlanManager

**新模块：** `core/plan_manager.py`

```
PlanManager
├── get_daily_plan(date) → DailyPlanResponse
│   └── 从 daily_tasks 表取当日题单，不存在则触发 _allocate_tasks()
├── mark_task(task_id, status) → DailyTask
│   └── 单题打卡（允许 7 天内补卡）
├── get_progress(days=7) → list[dict]
│   └── 近 N 天每日完成数统计
└── _allocate_tasks(plan_id, start_date, end_date)
    └── 按难度权值平均分配：难度越大 → 当天题目越少
```

**分配算法（`_allocate_tasks`）：**
1. 取 StudyPlan 中所有待分配题目，按 difficulty 降序排列
2. 计算每道题的"权值"：`weight = difficulty / 800`（800 分题 = 1 单位，2400 分题 = 3 单位）
3. 设定每天总权值上限：`daily_capacity = daily_count × 1.5`
4. 贪心分配：每天尽量填满容量，权值大的题优先排在前面
5. 写入 `daily_tasks` 表

### 5.4 OJAdapter 扩展

每个适配器新增方法：

```python
# adapters/base.py
@abstractmethod
def get_problem_url(self, problem_id: str) -> str:
    """返回题目的完整 URL"""
    ...

# CodeforcesAdapter:  f"https://codeforces.com/problemset/problem/{contestId}/{index}"
# LuoguAdapter:       f"https://www.luogu.com.cn/problem/{pid}"
# AtCoderAdapter:     f"https://atcoder.jp/contests/{contest}/tasks/{problem_id}"
# NowCoderAdapter:    f"https://ac.nowcoder.com/acm/contest/{cid}/{pid}"
```

### 5.5 VSCode Webview 组件树

```
DailyPlanPanel（根组件）
├── PlanHeader       — 日期、进度概览（已完成 X/Y）、刷新按钮
├── FilterBar        — OJ / 状态 / 难度 筛选
├── ProgressChart    — 本周打卡热力图（mini 色块，可折叠）
└── ProblemList      — 可滚动题目列表
    └── ProblemCard × N
        ├── 编号 + OJ 图标
        ├── 题名（可点击 → vscode.env.openExternal 跳转 OJ）
        ├── 难度 Tag（颜色按难度分级）
        ├── 状态下拉框（未开始/尝试中/已AC/已放弃）
        └── 打卡确认按钮
```

### 5.6 Webview ↔ Extension Host 消息协议

| 方向 | 消息类型 | Payload |
|------|----------|---------|
| Webview → Host | `getDailyPlan` | `{ date }` |
| Host → Webview | `dailyPlanData` | `DailyPlanResponse` |
| Webview → Host | `markTask` | `{ taskId, status }` |
| Host → Webview | `taskUpdated` | `DailyTask` |
| Webview → Host | `openProblem` | `{ url }` |

### 5.7 CLI 扩展

```bash
acm daily                    # 今日题单（Rich 表格）
acm daily --date 2026-06-28  # 指定日期
acm daily mark <id> --status ac_completed  # 打卡
```

### 5.8 关键设计决策（ADR）

| # | 决策 | 理由 | 代价 |
|---|------|------|------|
| ADR-001 | 新增 `daily_tasks` 独立表 | 需要按日期查询和单题状态更新，JSON 字段无法高效索引和原子更新 | 多一张表，需要分配逻辑 |
| ADR-002 | PlanManager 独立模块 | Database 管持久化，PlanManager 管计划执行逻辑，单一职责 | 多一个文件 |
| ADR-003 | 题目链接由 OJAdapter 生成 | 各 OJ URL 规则不同，适配器已掌握平台知识 | 每个适配器多一个方法 |
| ADR-004 | Webview 用原生 HTML/TS | 项目体量小，框架构建链增加复杂度，原生足够 | 复杂交互时模板代码稍多 |
| ADR-005 | 按难度权值分配每日题量 | 难题需要更多时间，同一天塞 3 道 2400 分题不合理 | 分配算法需要调参 |
| ADR-006 | 7 天补卡窗口 | 兼顾灵活性（偶尔忘记）和计划严肃性（太久远的补卡无意义） | 需要在 mark_task 中校验日期范围 |
| ADR-007 | 计划生成 → 预览确认 → 导入（方案 B） | 用户可审查和编辑 AI 生成的题单，避免不合适的题目被直接导入 | 多一个确认步骤；需要草稿临时存储和编辑接口 |
| ADR-008 | 智能去重而非全量排除已 AC 题 | 简单排除浪费温习机会；但无限制推荐会让用户烦躁 | 需维护 `_filter_reusable_problems` 规则；阈值可能需要用户可调 |
| ADR-009 | 去重规则：AC 率 < 70% + 距上次 AC > 60 天 + 提交 ≥ 3 次 | 三个条件联合筛选出"曾挣扎过、已生疏、值得温习"的题 | 边界 case：一道题 2 次提交 100% AC 但用户其实没掌握——会被过滤掉 |
| ADR-010 | LuoguAdapter 基于 `luogu-api-docs` 使用官方 REST API | 官方 API 比爬虫更稳定、更高效；`luogu-api-docs` 项目活跃（★99，194 commits） | 需适配洛谷 API 变更（但有文档追踪）；不能包含 `python-requests` UA |

---