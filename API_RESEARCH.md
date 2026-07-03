
# ACM Tracker 项目架构文档（API 调研增补）

> 本文档是 `ARCHITECTURE.md` 的 API 调研增补，基于四大 OJ 平台 API 实际调研结果编写。
>
> **调研时间：** 2026-06-28
>
> **调研结论速览：**
> | OJ 平台 | API 类型 | 推荐数据源 | 稳定性 | 备注 |
> |---------|---------|-----------|--------|------|
> | Codeforces | 官方 REST API | `codeforces.com/api` | ⭐⭐⭐⭐⭐ | 唯一官方来源，无需修改 |
> | 洛谷 (Luogu) | 逆向工程 API | `luogu-api-docs` 项目 | ⭐⭐⭐⭐ | 2026-05 大重构后已修复 |
> | AtCoder | 第三方 API | `kenkoooo.com` V3 | ⭐⭐⭐⭐ | 时间分页，500条/页 |
> | 牛客 (NowCoder) | 逆向工程 AJAX | 内部 JSON 端点 | ⭐⭐ | 风险最高，需替代爬虫 |

---

## A. 各平台 API 详解

### A.1 Codeforces — 官方 REST API ✅

**API 类型：** 官方 REST API  
**文档地址：** https://codeforces.com/apiHelp  
**稳定性：** ⭐⭐⭐⭐⭐ 最高  
**速率限制：** ~1 req/2sec（软限制，无需 API Key）  
**认证：** 无需认证（公开数据）

#### 核心端点

| 功能 | 端点 | 说明 |
|------|------|------|
| 获取用户提交记录 | `GET /user.status?handle={handle}` | 包含 verdict、problem、timeConsumed、memoryConsumed |
| 获取题目列表 | `GET /problemset.problems` | 含 rating、tags，支持分页 |
| 获取用户信息 | `GET /user.info?handles={handle}` | rating、rank、contribution |
| 获取用户 rating | `GET /user.rating?handle={handle}` | 历史 rating 变化 |

**架构状态：** 已有实现完善，无需修改。

---

### A.2 洛谷 (Luogu) — 逆向工程 API ⚠️

**API 类型：** 逆向工程（无官方 API）  
**文档来源：** `0f-0b/luogu-api-docs`（GitHub，194 commits，持续更新中，2026-06-15 最后更新）  
**稳定性：** ⭐⭐⭐⭐ 较好  
**速率限制：** 软限制 ~1-2 req/sec，无明确文档  
**认证：** 部分端点需 Cookie（`_uid` + `__client_id`）

#### 重要发现

1. **2026-05 大重构风险已过**：洛谷在 2026-05-24~31 进行了前端大重构，部分 API 损坏。`luogu-api-docs` 已在 2026-06 陆续修复，当前状态可用。

2. **Python SDK 可用**：`NekoOS-Group/luogu-api-python`（2026-04 更新）是专用 Python SDK，可参考其实现。

3. **User-Agent 限制**：
   - ❌ 不能包含 `python-requests`（不区分大小写）
   - ❌ 不能以 `Mozilla/` 开头
   - ✅ 建议：`MyApp/1.0 (contact@example.com)` 或自定义 UA

4. **CSRF Token**：提交操作需 CSRF Token，**必须从对应题目页面获取**（不是主页）。

#### 核心端点

| 功能 | 端点 | 方法 | 必需 Header | 备注 |
|------|------|------|-------------|------|
| 获取用户提交记录 | `/record/list?uid={uid}&page={page}` | GET | `x-luogu-type: content-only` | 核心端点 |
| 获取题目详情 | `/problem/{pid}` | GET | `x-lentille-request: content-only` | — |
| 获取用户信息 | `/user/{uid}` | GET | `x-lentille-request: content-only` | — |
| 搜索用户 | `/api/user/search?keyword={keyword}` | GET | 无 | 替代手动 UID 解析 |
| 获取用户_rating | `/api/rating/elo` | GET | 无 | 🆕 新增 |
| 获取练习统计 | `/user/{uid}/practice` | GET | `x-lentille-request: content-only` | 🆕 新增 |

#### 提交记录状态映射（luogu-api-docs → Verdict）

| 洛谷状态码 | 含义 | Verdict |
|-----------|------|---------|
| 12 | Accepted | AC |
| 14 | Wrong Answer | WA |
| 15 | Time Limit Exceeded | TLE |
| 16 | Time Limit Exceeded (more) | TLE |
| 17 | Memory Limit Exceeded | MLE |
| 18 | Memory Limit Exceeded (more) | MLE |
| 19 | Runtime Error | RE |
| 20 | Runtime Error (more) | RE |
| 21 | Compile Error | CE |
| 其他 | — | UNKNOWN |

---

### A.3 AtCoder — 第三方 API (kenkoooo.com) ⚠️

**API 类型：** 社区维护的第三方 API（非官方）  
**文档地址：** https://github.com/kenkoooo/AtCoderProblems/blob/master/docs/apidiff.md  
**稳定性：** ⭐⭐⭐⭐ 较好（被广泛使用）  
**速率限制：** 推荐 ~1 req/sec  
**认证：** 无需认证

#### 核心端点

| 功能 | 端点 | 说明 |
|------|------|------|
| 用户提交记录 | `GET /v3/user/submissions?user={handle}` | 时间分页，500 条/次 |
| 用户 AC 记录 | `GET /v3/user/ac?user={handle}` | 仅 AC 的去重列表 |
| 题目列表 + 难度 | `GET /v3/problems` | 含 difficulty 字段 |
| 指定时间范围提交 | `GET /v3/user/submissions?user={h}&from_second={ts}` | 分页续接 |
| Contest 列表 | `GET /v2/contest` | 获取比赛列表 |

#### 分页机制（关键）

kenkoooo.com 使用**时间戳分页**，不支持 offset/page 模式：

```
第 1 页: GET .../user/submissions?user={h}
第 2 页: GET .../user/submissions?user={h}&from_second={last_epoch_second}
第 3 页: GET .../user/submissions?user={h}&from_second={last_epoch_second+1}
... 循环直到无新数据
```

**⚠️ 注意：** 需要循环获取全部历史提交，不能只取 500 条。

**难度映射：** `difficulty × 0.8 + 800`（统一映射为 CF rating 体系，不变）

---

### A.4 牛客 (NowCoder) — 逆向工程 AJAX API ❌（风险最高）

**API 类型：** 逆向工程内部 AJAX（无官方 API）  
**稳定性：** ⭐⭐ 风险较高  
**速率限制：** 未文档化，社区经验 ~1 req/sec  
**认证：** 无需认证（公开数据），提交代码需 Cookie

#### 现状：爬虫 → 迁移到 JSON 端点

当前架构使用 HTML 爬虫，**最脆弱**。经调研，牛客内部有 JSON API 可替代：

| 功能 | JSON 端点 | 方法 | 备注 |
|------|----------|------|------|
| 比赛排行榜 | `https://ac.nowcoder.com/acm-heavy/acm/contest/real-time-rank-data?id={cid}&limit=0&page=1` | GET | 完整排名+每题状态 |
| 提交状态列表 | `https://ac.nowcoder.com/acm-heavy/acm/contest/status-list?id={cid}&searchUserName={name}` | GET | 🆕 核心端点 |
| 用户参赛历史 | `https://ac.nowcoder.com/acm-heavy/acm/contest/profile/contest-joined-history?uid={uid}` | GET | 🆕 比赛ID收集 |
| 比赛日历 | `https://ac.nowcoder.com/acm/calendar/contest?month={YYYY-MM}` | GET | 🆕 |
| 比赛搜索 | `https://ac.nowcoder.com/acm-heavy/acm/contest/search-detail?searchName={name}` | GET | 获取比赛 ID |

#### 牛客提交同步路径

牛客没有全量提交记录 API（类似 CF 的 `/user.status`），需分步获取：

```
1. 获取用户 uid
       ↓
2. 通过 /contest/profile/contest-joined-history 获取用户参与的所有比赛 ID
       ↓
3. 遍历每场比赛，通过 /status-list 获取该用户的提交记录
       ↓
4. 合并所有提交记录
```

**请求参数规范：** 所有请求需包含 `token=`（空值）和 `_={unix_ts_ms}`。

#### 参考实现

| 项目 | 语言 | GitHub |
|------|------|--------|
| `Floating-Ocean/OBot-ACM` | Python | ACM 排行 + 用户信息（最完整参考） |
| `xcpcio/board-spider` | Python | 实时排名数据 |
| `RealClearwave/Nowcoder_RemoteJudge` | Python | 提交代码（需 Cookie） |

---

## B. 新增 ADR

| # | 决策 | 理由 | 代价 |
|---|------|------|------|
| ADR-011 | NowCoder 适配器从爬虫迁移到 JSON AJAX 端点 | HTML 爬虫最脆弱；JSON 端点返回结构化数据更稳定；参考 OBot-ACM 等实现 | 无全量提交 API，需先获取比赛 ID 列表再逐比赛同步 |
| ADR-012 | 牛客提交同步需要比赛 ID 收集步骤 | 记录分散在各比赛中 | 比赛多则同步慢，需增量同步策略 |
| ADR-013 | 牛客请求固定包含 `token=` 和 `_={timestamp_ms}` | 这是内部 AJAX 固定格式 | 无（兼容行为） |
| ADR-014 | AtCoder 同步需循环获取全部历史提交（时间分页） | 每次最多 500 条 | 循环调用可能较慢 |

---

## C. API 端点速查表

### Codeforces

```
基础 URL: https://codeforces.com/api
GET /user.status?handle={handle}          # 提交记录
GET /problemset.problems                   # 题目列表
GET /user.info?handles={handle}            # 用户信息
GET /user.rating?handle={handle}          # Rating 历史
Rate: ~1 req/2sec  Auth: 无
```

### 洛谷 (Luogu)

```
基础 URL: https://www.luogu.com.cn
GET /record/list?uid={uid}&page={page}    # 提交记录 (需 x-luogu-type: content-only)
GET /problem/{pid}                         # 题目详情 (需 x-lentille-request: content-only)
GET /user/{uid}                            # 用户信息 (需 x-lentille-request: content-only)
GET /api/user/search?keyword={keyword}    # 用户搜索
UA限制: 不能包含"python-requests"，不能以"Mozilla/"开头
```

### AtCoder

```
基础 URL: https://kenkoooo.com/atcoder-api/v3
GET /user/submissions?user={handle}              # 提交记录 (500条/页)
GET /user/submissions?user={h}&from_second={ts}  # 分页续接
GET /user/ac?user={handle}                       # 仅 AC 记录
GET /problems                                    # 题目列表 (含 difficulty)
Rate: ~1 req/sec  Auth: 无
难度映射: difficulty × 0.8 + 800 (统一为 CF rating)
```

### 牛客 (NowCoder)

```
基础 URL: https://ac.nowcoder.com
GET /acm-heavy/acm/contest/real-time-rank-data?id={cid}&limit=0&page=1  # 排行
GET /acm-heavy/acm/contest/status-list?id={cid}&searchUserName={name}   # 提交
GET /acm-heavy/acm/contest/profile/contest-joined-history?uid={uid}     # 参赛历史
GET /acm/calendar/contest?month={YYYY-MM}                               # 日历
⚠️ 所有请求需: token= (空值) + _={unix_ts_ms}
⚠️ 获取用户提交: ① 获取比赛 ID 列表 ② 逐比赛获取提交
```

---

## D. 行动项（按优先级）

### P0 — 必须修复
- [ ] **ADR-011**: NowCoderAdapter 从爬虫迁移到 JSON AJAX 端点（最高风险项）
- [ ] **ADR-014**: AtCoder 同步循环获取全部历史提交

### P1 — 重要改进
- [ ] **ADR-012**: NowCoder 补充比赛 ID 收集逻辑
- [ ] **ADR-013**: 牛客请求添加固定参数
- [ ] 验证 LuoguAdapter 使用了 `/record/list` 端点

### P2 — 可选优化
- [ ] 参考 `NekoOS-Group/luogu-api-python` 完善 LuoguAdapter
- [ ] 参考 `OBot-ACM` 重构 NowCoder 适配器
- [ ] Luogu 增加 `/api/rating/elo` 获取 rating 历史数据
