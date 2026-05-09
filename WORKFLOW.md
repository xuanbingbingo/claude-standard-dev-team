# 11 阶段工作流详解

这一份是给"想理解这套 agent 怎么协作"的人看的。如果你只是想用它跑项目，看 [README.md](README.md) + [INSTALL.md](INSTALL.md) 就够了。

---

## 一、Phase 调度全图

```
Phase 0   orchestrator 创建目录                          （10 秒）
   │
   ▼
Phase 1   ┌─→ Task → product-manager        →  PRD.md
          │
          ⏸ ───── 人工检查点 #1 ──────
          │   等用户确认"继续"
          ▼
Phase 2   ┌─→ Task → software-architect    →  API_CONTRACT.md
          │                                    DB_SCHEMA.md
          │                                    TECH_SPEC.md
          ⏸ ───── 人工检查点 #2 ──────
          │   等用户确认"继续"（最关键的人工节点）
          ▼
Phase 2.5 ┌─→ Task → ui-designer           →  DESIGN_SYSTEM.md
          │                                    variables.css
          ▼
Phase 3   orchestrator 自己读 API_CONTRACT  →  backend-tasklist.md
          + DB_SCHEMA 拆任务清单               frontend-tasklist.md
          ▼
Phase 4   ┌─→ Task → database-optimizer    →  migrations/
                                               start.sh
          ▼
Phase 5   ┌─→ Task → backend-architect    │  ┐
          │   ↓                            │  │  Dev-QA Loop
          │   Task → ev-collector 验证     │  │  逐任务循环
          │   ↓                            │  │  （详见下节）
          │   PASS / FAIL 决策            │  ┘
          ▼
Phase 6   ┌─→ Task → frontend-developer  │  ┐  Dev-QA Loop
          │   ↓                            │  │  逐任务循环
          │   Task → ev-collector         │  │
          │   ↓                            │  │
          │   PASS / FAIL                 │  ┘
          ▼
Phase 7   ┌─→ Task → security-engineer   →  SECURITY_REPORT.md
          ▼
Phase 8   ┌─→ Task → code-reviewer       →  REVIEW_REPORT.md
          ▼
Phase 9   ┌─→ Task → devops-automator    →  Dockerfile / compose
                                              + 部署路径前缀检查
          ▼
Phase 10  ┌─→ Task → reality-checker      →  READY 或 NEEDS WORK
          │                                  （默认 NEEDS WORK，
          │                                   要压倒性证据才 READY）
          ▼
Phase 11  ┌─→ Task → technical-writer     →  README.md + API_DOC.md
          ▼
        完工 ✅
```

---

## 二、Dev-QA Loop（核心创新）

这是这套 orchestrator 比一般 agent 编排高级的地方——**每个任务都有任务级 QA 闭环**。

```
读 backend-tasklist.md
    │
    ▼
FOR 每个 [ ] 任务:
    │
    ├─→ STEP 1: Task → backend-architect
    │             • Read API_CONTRACT.md
    │             • Read DB_SCHEMA.md
    │             • 实现该任务
    │             • 返回："已实现 POST /api/v1/auth/login"
    │
    ├─→ STEP 2: Task → testing-evidence-collector（QA 验证）
    │             • Read API_CONTRACT 中该接口定义
    │             • 扫描刚实现的代码文件
    │             • 检查：路径/字段名/错误处理
    │             • 返回：PASS 或 FAIL + 原因
    │
    └─→ STEP 3: 决策
             ├─ PASS  → 任务标 [x] → 下一任务
             ├─ FAIL（重试 < 3）→ QA 反馈带回去给 backend-architect 重做
             └─ FAIL（重试 ≥ 3）→ 暂停，向用户报告卡点
    │
    ▼
ALL 任务 PASS 后:
    检查 BACKEND_STATUS.md ISSUES 章节
    │
    └─ 有未解决 → 打回 software-architect 改契约 → 重跑受影响任务
```

**关键洞察**：QA agent 是**单独 LLM session**——它不知道 backend-architect 内部怎么写的，只对照 API_CONTRACT 看产出代码。这种"评审者不是实现者"的设计，比让一个 agent 自己写自己测可靠 10 倍。

---

## 三、打回机制（重试树）

```
任意 Phase 出问题：
                        ┌─────────────────────────────────┐
                        │  问题分类                        │
                        └────────────┬────────────────────┘
                                     │
        ┌────────────────────────────┼────────────────────────────┐
        │                            │                            │
        ▼                            ▼                            ▼
   契约有问题                    实现有问题                    评审有问题
   (DB_ISSUES /                 (任务级 QA FAIL)              (REVIEW MUST FIX
    BACKEND ISSUES)              (安全高危)                    reality-checker
        │                            │                          NEEDS WORK)
        │                            │                            │
        ▼                            ▼                            ▼
   打回 software-architect       打回对应实现 agent           打回对应 agent
   修改契约 → 重跑后续           重做该任务                   修复
   (重试上限 2)                  (重试上限 3/任务)             (重试上限 1-2)
        │                            │                            │
        └────────────────────────────┴────────────────────────────┘
                                     │
                                     ▼
                              超限 → 暂停 → 向用户报告卡点
```

### 最严苛的零容忍规则

- ❌ **前端硬编码** `/api/...` 路径（不用 VITE_API_BASE 环境变量）→ 必须打回，3 次重试上限
- ❌ **Phase 9 部署路径前缀检查** FAIL → 必须打回 frontend-developer 重新验证

这两条是在多次部署翻车后总结进 orchestrator 的。

---

## 四、人工检查点（只 2 处暂停）

整个 11 阶段只在 2 个地方让用户介入：

```
Phase 1 完成后  ⏸  展示 PRD 功能列表 F01/F02/...
                  等用户确认"继续"
                  （目的：在生成 API 契约前最后确认范围，
                  否则 Phase 2 之后改一个字段就要重跑 Phase 4-6）

Phase 2 完成后  ⏸  展示 API 接口列表 + 数据库表结构摘要
                  等用户确认"继续"
                  （目的：契约一旦确认，所有 agent 都依赖它；
                  确认后任何接口字段名变更都是大事故）

之外全部自动跑，不打扰用户。
```

**为什么不在 Phase 5/6/7/8 等地方暂停？**

因为这些阶段有自动 Dev-QA Loop 和打回机制，能自愈。让用户每个任务都确认会很烦。**只在"会引发雪崩式重跑"的关键节点才打断用户**。

---

## 五、并行 vs 串行

```
强串行依赖（必须按顺序）：
  Phase 1（PRD）→ Phase 2（API 契约）→ Phase 4（DB 实现）
  Phase 4 → Phase 5（后端，依赖 DB）
  Phase 2 → Phase 6（前端，依赖契约）
  Phase 5 → Phase 9（DevOps，依赖完整代码）

可并行的潜在机会：
  Phase 5（后端）和 Phase 6（前端）理论上可并行
  → 但 orchestrator 选择了串行（先后端再前端）
  → 因为：前端需要后端 API 真跑通了才能联调，
    并行会让 ev-collector 没有真实接口可测试

  Phase 7（安全）和 Phase 8（review）理论上可并行
  → 但 orchestrator 选择了串行
  → 因为：Phase 7 修完安全问题后可能改了代码，
    Phase 8 在并行时可能 review 老代码
```

**设计哲学**：这个 orchestrator 选择"质量 > 速度"，宁可慢一点串行跑，也不让两个 agent 同时改代码。

---

## 六、典型时序举例

用户说："使用标准团队开发一个 todo app"会发生什么：

```
T=0:00    用户："使用标准团队开发一个 todo app"
T=0:00    主对话 → Task → orchestrator 启动

T=0:01    orchestrator → Bash mkdir -p docs project-tasks
T=0:01    orchestrator → Task → product-manager
T=0:08    pm 返回 "PRD.md 已写"

T=0:08    ⏸ 主对话向用户展示 PRD 功能列表，等确认
T=2:00    用户："继续"

T=2:00    orchestrator → Task → software-architect
T=2:15    sw-arch 返回 "API_CONTRACT/DB_SCHEMA/TECH_SPEC 已写"

T=2:15    ⏸ 主对话展示 API 接口列表 + 表结构，等确认
T=5:00    用户："继续"

T=5:00    orchestrator → Task → ui-designer
T=5:08    ui 返回 "DESIGN_SYSTEM.md + variables.css 已写"

T=5:08    orchestrator 自己拆任务清单（不调 agent）

T=5:09    orchestrator → Task → database-optimizer
T=5:15    db 返回 "migrations/ + start.sh 已写"

T=5:15    Phase 5 后端 Loop 开始
          FOR 任务 #1: POST /api/v1/auth/login
            T=5:15  → Task → backend-architect
            T=5:25  ba 返回 "已实现"
            T=5:25  → Task → ev-collector
            T=5:28  ev 返回 "PASS"
            T=5:28  任务标 [x]
          FOR 任务 #2: GET /api/v1/users
            T=5:28  → Task → backend-architect
            T=5:35  ba 返回 "已实现，但发现 schema 缺一个字段"
            T=5:35  写入 BACKEND_STATUS.md ISSUES
            T=5:35  → Task → ev-collector
            T=5:38  ev 返回 "FAIL（字段缺失）"
            T=5:38  打回 software-architect 改契约
            T=5:42  契约修复
            T=5:42  → Task → backend-architect 重做
            T=5:50  ba 返回 "已实现"
            T=5:50  → Task → ev-collector
            T=5:52  ev 返回 "PASS"
          ... （所有任务循环完毕）

T=8:00    Phase 5 全部 PASS

T=8:00    Phase 6 前端 Loop 开始（同样的 Dev-QA 循环）
T=12:00   Phase 6 全部 PASS

T=12:00   Phase 7 → security-engineer  →  SECURITY_REPORT.md
T=12:30   Phase 8 → code-reviewer       →  REVIEW_REPORT.md
T=13:00   Phase 9 → devops-automator    →  Dockerfile + 路径检查
T=13:15   Phase 10 → reality-checker    →  READY
T=13:30   Phase 11 → technical-writer   →  README + API_DOC

T=13:30   orchestrator 给主对话发最终汇报
T=13:30   主对话给用户看汇报
```

整个过程**主对话只看到几次 orchestrator 的进度汇报**，真正干活的 12 个团队成员各自在独立 session 里跑（总指挥 orchestrator 不写代码只调度）。**主对话的 token 消耗极低**——几千 token 就跑完了一个完整项目。

---

## 七、各 Phase 详解

### Phase 0 - 项目初始化

orchestrator 直接执行：

```
mkdir -p docs project-tasks
```

不调用任何子 agent，10 秒搞定。

### Phase 1 - PRD（product-manager）

输入：用户的原始需求（一句话也行）  
输出：`docs/PRD.md`，包含：
- 用户故事
- 功能清单（F01/F02/...）
- 验收标准
- 非功能需求（性能/安全/兼容性）
- MVP 范围划定

**人工检查点 #1**：用户在这里看 PRD 决定是否要砍/加功能。

### Phase 2 - 技术契约（software-architect）⭐ 最关键

输入：PRD.md  
输出：
- `docs/API_CONTRACT.md` — 所有 REST/RPC 接口定义（路径/方法/请求/响应/错误码）
- `docs/DB_SCHEMA.md` — 表结构 + 字段类型 + 索引 + 外键
- `docs/TECH_SPEC.md` — 技术选型 + 架构决策

**这是整个项目最关键的一份文档。后面 9 个 Phase 全都依赖它。**

**人工检查点 #2**：用户在这里看接口列表 + 表结构。一旦确认，任何字段名变更都是大事故。

### Phase 2.5 - 设计系统（ui-designer）

输入：PRD.md + TECH_SPEC.md  
输出：
- `docs/DESIGN_SYSTEM.md` — 颜色 / 间距 / 字号 / 组件规范
- `frontend/styles/variables.css` — 设计 token CSS 变量

### Phase 3 - 任务拆解（orchestrator 自己干）

orchestrator 读 API_CONTRACT + DB_SCHEMA，自己拆出：
- `project-tasks/backend-tasklist.md` — 后端任务清单（每个接口一个任务）
- `project-tasks/frontend-tasklist.md` — 前端任务清单（每个页面一个任务）

不调 agent，只是把契约转成 checkbox 列表。

### Phase 4 - 数据库（database-optimizer）

输入：DB_SCHEMA.md  
输出：
- `backend/migrations/` — 迁移文件（按时间戳命名）
- `backend/start.sh` — 启动脚本（含 migration 自动执行）

### Phase 5 - 后端实现（backend-architect + ev-collector Loop）

按 backend-tasklist 逐个任务跑 Dev-QA Loop（详见上文第二节）。

### Phase 6 - 前端实现（frontend-developer + ev-collector Loop）

按 frontend-tasklist 逐个任务跑 Dev-QA Loop。**前端零容忍硬编码 API 路径**——必须用 `import.meta.env.VITE_API_BASE` 之类的环境变量。

### Phase 7 - 安全审计（security-engineer）

输入：完整代码库  
输出：`docs/SECURITY_REPORT.md`，包含：
- OWASP Top 10 检查
- 鉴权逻辑评估
- 数据脱敏检查
- 高危漏洞清单（必须修）
- 中低危建议（可暂缓）

高危项必须打回相应实现 agent 修复。

### Phase 8 - 代码评审（code-reviewer）

输入：完整代码库  
输出：`docs/REVIEW_REPORT.md`，分类：
- MUST FIX — 阻塞上线
- SHOULD FIX — 建议改
- NICE TO HAVE — 可选

MUST FIX 必须打回，其他可上线后慢慢改。

### Phase 9 - 部署（devops-automator）

输入：完整代码库  
输出：
- `Dockerfile`
- `docker-compose.yml`
- CI/CD 配置（如适用）
- **basePath 检查报告**：扫所有前端代码确认没有硬编码 `/api/...`

> **延伸：从"产出 Dockerfile"到"真正上线"（可选）**
>
> Phase 9 跑完只是产出了**部署素材**（Dockerfile / compose）。如果你要把它真正部署到一台云服务器跑起来，仓库 `agents/` 下额外提供了基建层 3 个 agent：
>
> 1. `infra-bootstrap-agent` — 服务器一次性初始化（gateway-net / nginx-gateway / infra-mysql）
> 2. `app-deploy-agent` — 按仓库根目录的 `deploy.yaml` 把应用部署上去
> 3. `deploy-yaml-schema` — `deploy.yaml` 字段契约
>
> 这 3 个 agent 不在 11 Phase 主链路里，由你**手动触发**。详见 README "进阶：完整部署链路"小节。基建层带作者私有约定，按需启用。

### Phase 10 - 最终验收（reality-checker）

reality-checker 是个**疑心病**——默认判决是"NEEDS WORK"。它会：
- 对照原始 PRD 看每个功能是否真的实现了
- 检查所有契约文件是否还和最终代码一致
- 抽样跑代码看是否真的能起来

只有**压倒性证据**才会判 READY。这是上线前最后一道关。

### Phase 11 - 文档（technical-writer）

输入：完整代码库 + 所有契约文档  
输出：
- `README.md` — 项目介绍 + 快速上手 + 架构说明
- `docs/API_DOC.md` — 给外部使用者看的 API 文档（不是给 backend-architect 的契约）
- `docs/TUTORIAL.md`（可选）— 教程

---

## 八、设计哲学总结

| 设计原则 | 体现在哪 |
|--|--|
| **契约第一** | Phase 2 是最关键节点，后面 9 个 Phase 全都依赖契约 |
| **评审者 ≠ 实现者** | ev-collector 独立验证 backend-architect/frontend-developer 产出 |
| **打回有上限** | 重试 3 次还不行就暂停问用户，不死循环 |
| **人工介入点要少** | 只在 Phase 1/2 暂停，避免每步都打扰用户 |
| **职责单一** | 每个 agent 只干一类事，互不交叉 |
| **质量 > 速度** | 宁可串行也不并行（前后端不能同时改代码） |
| **零容忍硬编码** | API 路径硬编码必须打回（来自部署翻车的真实教训） |

---

读完想动手试试？回到 [INSTALL.md](INSTALL.md) 装上之后说一句"使用标准团队帮我做 X"就开始了。
