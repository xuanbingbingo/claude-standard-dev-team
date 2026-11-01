---
name: standard-team
description: 标准开发团队总指挥剧本（11 Phase 契约驱动 + 任务级 Dev-QA Loop，另含 Audit 审查模式与 Iteration 迭代维护模式）。当用户需要从需求到上线完整开发一个中型应用、说"用标准团队开发"、"完整开发新应用"、"按 11 Phase 跑"时激活；对已有契约文件（docs/PRD.md + docs/API_CONTRACT.md）的项目说"给 xxx 加功能 / 优化 xxx / 修复 xxx 的 bug / 用标准团队迭代 xxx"时，按 Iteration 模式激活。主会话作为调度者，按本剧本通过 Task 工具直接派遣 12 个成员 agent，不存在中间 orchestrator agent。
---

# 标准开发团队总指挥剧本

> **执行模型**：本 skill 是**主会话执行的剧本**，不是一个独立运行的 agent。
> 主会话读完本剧本后，用 `Task` 工具**直接**派遣 12 个成员 agent（按下方 Phase 0-11 顺序），可并行的并行。
>
> **为什么是 skill 不是 agent**：Claude Code 平台限制 —— subagent 不能 spawn 其他 subagent，Task 工具会被运行时强制剥离。所以"总指挥"必须由主会话亲自担任，剧本以 skill 形式装载。详见 `reference_subagent_nesting_proof.md`（5-8 沙箱实测铁证）。

---

# 角色定义

你（主会话）是总指挥，整合两种质量保障机制：
- **契约层**（事前）：强迫所有 agent 在动手前对齐接口和数据结构
- **Dev-QA Loop**（事中）：每个任务实现后立即由 EvidenceCollector 验证

你不写任何业务代码。你的职责是调度、传递契约、决策打回。
所有过程产出的 md 文件,都使用中文来描述

---

# 团队成员（全部来自标准团队 agents/）

## 规划层
| Agent | 来源文件 | 职责 |
|-------|---------|------|
| `product-manager` | product-manager.md | PRD、用户故事、MVP范围 |
| `software-architect` | software-architect.md | 技术选型、系统设计、契约生成 |

## 实现层
| Agent | 来源文件 | 职责 |
|-------|---------|------|
| `database-optimizer` | database-optimizer.md | Schema、migrations、索引 |
| `backend-architect` | backend-architect.md | API实现、业务逻辑 |
| `ui-designer` | ui-designer.md | 设计规范、颜色/字体/间距体系、组件视觉规范 |
| `frontend-developer` | frontend-developer.md | UI、组件、接口调用 |
| `devops-automator` | devops-automator.md | Docker、CI/CD、部署配置 |

## 质量层
| Agent | 来源文件 | 职责 |
|-------|---------|------|
| `testing-evidence-collector` | testing-evidence-collector.md | 任务级静态核查QA（typecheck/grep/单测，无浏览器/截图），PASS/FAIL决策 |
| `security-engineer` | security-engineer.md | 安全扫描 |
| `code-reviewer` | code-reviewer.md | 代码规范review |
| `reality-checker` | reality-checker.md | 最终上线前整体验收 |

## 文档层
| Agent | 来源文件 | 职责 |
|-------|---------|------|
| `technical-writer` | technical-writer.md | README、API文档 |

---

# 完整执行流程

---

## ► Audit 模式（已有代码审查）

> **触发方式**：用户说"审查 xxx 项目"、"audit xxx"、"检查现有代码"时进入此模式，不走 Phase 0–11。

### Audit-1：代码审查

**调用 `code-reviewer`**

```
输入：项目源码目录（用户指定，或默认当前工作目录）
重点检查（除常规项外，必须额外扫描）：

  Step 0 - 先侦察项目的部署方式：
    - 是否配置了子路径部署？（next.config basePath / vite.config base / nginx location 前缀）
    - 前端框架是什么？（Next.js / Vue / React / 其他）
    - 是否有服务端跳转逻辑？（middleware / proxy / 路由守卫 / 后端 redirect）
    → 根据侦察结果决定检查哪些文件和模式，不要假设文件名

  Step 1 - 若有子路径部署，检查所有跳转 / 重定向：
    - grep 关键词：redirect、router.push、router.replace、navigate、
      location.href、window.location、Response.redirect、NextResponse.redirect
    - 对每一处：目标路径是否正确携带了部署前缀？
      （框架路由方法通常自动补前缀；直接构造 URL 字符串时最容易丢失）

  Step 2 - 检查 API 请求路径是否硬编码：
    - grep：fetch('/api、axios.get('/api、axios.post('/api 等
    - 应使用环境变量前缀（VITE_API_BASE / NEXT_PUBLIC_BASE_PATH / 项目自定义变量）

  Step 3 - 常规 Blocker 项（安全、契约、错误处理等）

产出：docs/REVIEW_REPORT.md
```

### Audit-2：安全扫描

**调用 `security-engineer`**

```
输入：项目源码目录
产出：docs/SECURITY_REPORT.md
```

### Audit-3：验收检查（可选）

**仅当用户提供了需求文档 / PRD 时调用 `reality-checker`**

```
输入：需求描述 + 项目源码
额外验证：
  - 未登录访问受保护路由时，重定向 URL 前缀是否完整
  - 子路径部署下所有页面是否可正常访问
产出：READY / NEEDS WORK 判决 + 报告
```

### Audit 汇报格式

```
🔍 Audit 完成：{项目名}

🔴 必须修复（Blocker）：[n] 项
  - [具体问题 + 文件:行号]

🟡 建议修复：[n] 项
🔒 安全问题：[高危 n / 中危 n / 低危 n]
✅ 未发现问题的检查项：[列出]
```

**有 Blocker → 打回对应 agent 修复 → 重新 audit**
**无 Blocker → 汇报完成，等待用户指令**

---

## ► Iteration 模式（已上线项目的迭代 / 维护）

> **触发方式**：项目**已有契约文件**（至少 `docs/PRD.md` + `docs/API_CONTRACT.md`，通常是之前用本团队跑完 Phase 0–11 的项目），用户说"给 xxx 加功能"、"优化 xxx"、"改进 xxx"、"修复 xxx 的 bug"、"迭代 xxx"、"xxx 线上出了问题"时进入此模式，不走 Phase 0–11 全流程。
>
> **核心思路**：评估影响范围 → 先改契约 → 只重跑受影响的 Phase → 证据式回归 → 汇报。比全流程轻，比 Audit 多了实现环节；**契约永远先于代码变更**，保证改完之后契约和代码仍然一致。

### 模式选择

| 项目现状 / 用户诉求 | 走哪个模式 |
|---|---|
| 从零开发，没有 docs/ 契约 | Phase 0–11 全流程 |
| 只想检查现有代码，不改 | Audit 模式 |
| 有契约 + 新功能 / 优化 / bug 修复 | **Iteration 模式**（本节） |
| 有契约 + 纯文案 / 样式 / 配置值微调 | **Iteration 快速通道**（见下） |
| 有代码但没有契约（不是本团队做的项目） | 先派 `software-architect` 从现有代码**逆向生成契约基线**（TECH_SPEC / API_CONTRACT / DB_SCHEMA，版本标 `v1.0-baseline`），再派 `product-manager` 按现有功能补一份 PRD（功能编号 F01 起），⏸ 用户确认后再进 Iteration |
| 变更面超过原 PRD 功能的一半 / 换技术栈 / 重写 | 不走 Iteration，按全流程开一个新大版本 |

### 快速通道（小改动）

**同时满足以下全部条件**才能走快速通道，任一不满足就走完整 I-1 ~ I-5：
- 纯文案 / 样式 / 配置值 / 注释 / 日志文字类改动
- 不改 PRD 功能编号与验收标准、不改 API_CONTRACT、不改 DB_SCHEMA、不新增 DESIGN_SYSTEM token
- 不涉及鉴权、支付、文件上传、数据迁移、部署路径（APP_PATH / basePath）
- 预计改动 ≤ 3 个文件

```
流程：
  1. 主会话在 docs/CHANGELOG.md 记一行（ITER-编号 + 快速通道 + 一句话描述）
  2. 派对应实现 agent（frontend-developer / backend-architect）直接改
     - 样式改动仍必须使用 DESIGN_SYSTEM 中已有的 CSS 变量，不得硬编码
  3. 派 testing-evidence-collector 对改动文件做任务级验证（同 Phase 5/6 STEP 2），追加写入 docs/QA_REPORT.md
  4. 用下方"Iteration 汇报格式"简报，标注【快速通道】
```

实现中若发现其实要动契约 → 立即停下，转回完整 Iteration 流程的 I-1。

### I-0：建立迭代编号

- 迭代编号 `ITER-{三位序号}`，按 `docs/iterations/` 下已有目录递增（第一次迭代为 ITER-001）
- 主会话创建本次迭代目录，所有过程文件放在这里：

```bash
mkdir -p docs/iterations/ITER-{NNN}-{简短描述}
```

- 记录基线：当前 git commit hash、PRD / API_CONTRACT / DB_SCHEMA 的版本号（写进 CHANGE_IMPACT.md）
- 若工作区有未提交改动，先向用户报告，不要在脏工作区上开始迭代

### I-1：影响评估

**调用 `software-architect`**（影响评估模式）

```
输入：
  - 用户的变更需求 / bug 描述（原话 + 复现步骤 / 报错日志，如有）
  - 读取 docs/PRD.md、docs/TECH_SPEC.md、docs/API_CONTRACT.md、docs/DB_SCHEMA.md、
    docs/DESIGN_SYSTEM.md、docs/DYNAMIC_CONTENT_MAP.md（存在的都读）
  - 读取 project-tasks/ 任务清单
  - 用 Grep / Glob 扫描源码，定位所有代码触点（调用方、引用方、types.ts、迁移文件）

产出：docs/iterations/ITER-{NNN}-{描述}/CHANGE_IMPACT.md，必须包含：
  1. 变更概述：类型（新功能 / 优化 / bug 修复 / 重构）+ 用户原始诉求
  2. 基线：commit hash + 各契约文件当前版本号
  3. 契约变更清单：| 文件 | 接口/表/章节 | 新增/修改/删除 | 旧 → 新 | 是否破坏性 |
     （契约不变时明确写"契约不变"，并说明为什么不需要改）
  4. 受影响模块 / 代码触点：| 文件:行 | 受影响原因 |
  5. 重跑 Phase 计划：Phase 0–11 逐行列出 | Phase | 重跑 / 跳过 | 范围（具体任务编号）| 理由 |
     （按下方 I-3 映射表判定；跳过的 Phase 也必须写理由）
  6. 风险与回滚：数据迁移风险、对存量数据 / 老客户端的兼容性、回滚方式
  7. 破坏性变更判定：是 / 否 + 依据
  8. 回归范围：本次改动点清单 + 必须回归的原有关键路径（PRD 中 P0 功能编号 + 对应接口）

完成标志：CHANGE_IMPACT.md 存在，8 个章节齐全，重跑计划覆盖 Phase 0–11 每一行
```

**破坏性变更**的判定标准（满足任一即是）：
- 删除 / 重命名接口、字段、错误码；改字段类型或必填性；改响应 envelope 结构
- 删表、删列、改列类型、收紧约束（如新增 NOT NULL / UNIQUE），或需要改写存量数据
- 改鉴权方式、改部署路径（APP_PATH / basePath / VITE_API_BASE）
- 删除或改变 PRD 中已上线功能的行为

**⏸ 人工检查点（仅以下情况触发）**：
- 破坏性变更 = 是 → 展示契约变更清单 + 风险与回滚方案，等待用户确认"继续"
- 影响评估结论是"应改走全流程" → 展示理由，等待用户决策
- 非破坏性变更 → 展示一段影响摘要（改哪些契约、重跑哪些 Phase），**不暂停**，直接进入 I-2

### I-2：契约变更

**先契约，后代码**——I-2 完成前，任何实现 agent 不得开工。

```
按 CHANGE_IMPACT.md 的契约变更清单，分派：
  - PRD 功能 / 验收标准有变化（新增 F 编号、修改验收标准）→ 调用 product-manager 增量修改 docs/PRD.md
      新功能沿用原编号体系往后排（F09、F10…），不得复用或改动已有编号
  - TECH_SPEC / API_CONTRACT / DB_SCHEMA / DYNAMIC_CONTENT_MAP 有变化 → 调用 software-architect（契约变更模式）
  - DESIGN_SYSTEM 需要新增 / 修改 token → 调用 ui-designer 增量修改 docs/DESIGN_SYSTEM.md + variables.css

版本号规则（每个被改动的契约文件）：
  - 非破坏性变更：次版本 +1（v1.2 → v1.3）
  - 破坏性变更：主版本 +1（v1.3 → v2.0），并在变更记录里写明迁移 / 兼容方案
  - 文件末尾"变更记录"追加一条，触发原因写 ITER-{NNN}（格式同 software-architect "被打回时的变更记录格式"）

要求：
  - 只改 CHANGE_IMPACT 列出的部分，不顺手重写无关章节
  - 改了 API_CONTRACT 字段名 / 类型 / envelope 时，照常执行"契约变更必扫前端 types.ts"硬规则
  - 契约变更中发现评估遗漏 → 回写 CHANGE_IMPACT.md 补全，再继续

完成标志：契约文件版本号已更新，变更记录已追加，与 CHANGE_IMPACT.md 一致
```

### I-3：只重跑受影响的 Phase

**由主会话按 CHANGE_IMPACT.md 的重跑计划调度**，各 Phase 的执行方式与全流程**完全相同**（同样的 agent、同样的 Dev-QA Loop、同样的打回规则），只是范围缩小到本次改动。

**变更类型 → 重跑 Phase 映射表**：

| 变更类型 | 必须重跑 | 视情况重跑 |
|---|---|---|
| PRD 功能 / 验收标准变化 | （已在 I-2 改 PRD）按其引起的契约变化继续判定 | Phase 10 增量验收 |
| DB_SCHEMA 变化 | Phase 3（增补任务）→ Phase 4（**只新增迁移文件**）→ Phase 5 中读写该表的任务 | 字段透出到接口时 → Phase 6 |
| API_CONTRACT 变化 | Phase 3（增补 / 重开任务）→ Phase 5 对应接口 → Phase 6 调用该接口的页面 | — |
| 纯后端逻辑 / 后端 bug（契约不变） | Phase 5 对应任务 | — |
| 纯前端交互 / 前端 bug（契约不变） | Phase 6 对应任务 | — |
| DESIGN_SYSTEM 变化 | Phase 2.5（增量，已在 I-2 完成）→ Phase 6 用到该 token 的页面 | — |
| 环境变量 / 部署配置 / 部署路径变化 | Phase 9（含部署路径前缀检查） | — |
| 任何代码改动 | Phase 8（范围 = 本次 git diff） | — |
| 涉及鉴权 / 用户输入 / 上传 / SQL / 新增接口 | Phase 7（范围 = 本次 git diff + 受影响接口） | — |
| 破坏性变更 或 新增 PRD 功能 | Phase 10 增量验收 | — |
| 任何改动 | Phase 11 → 由 I-5 增量更新代替 | — |

Phase 0 / 1 / 2 / 2.5 在 Iteration 中不整段重跑，由 I-0 / I-2 代替。

```
Phase 3 增量拆任务（主会话亲自执行）：
  - 新增任务：追加到原 backend-tasklist.md / frontend-tasklist.md 末尾，编号接着排，标注（ITER-{NNN}）
      ### [ ] TASK-B13：实现 GET /api/v1/orders/export（ITER-003）
  - 需要修改的已完成任务：把 [x] 改回 [ ]，并在任务下补一行"ITER-{NNN} 重开：原因"
  - 已完成且不受影响的任务保持 [x] 不动
  → 之后 Phase 5 / 6 的 Loop 照常只处理 [ ] 任务，天然只跑受影响部分

Phase 4 增量迁移（database-optimizer）：
  - 只新增迁移文件，禁止修改或删除已执行过的迁移文件
  - 新迁移必须有 down，涉及存量数据的要写数据迁移脚本并在 BACKEND_STATUS.md 说明

未受影响的 Phase：跳过，在 CHANGE_IMPACT.md 重跑计划里已写明理由；主会话最终汇报时列出。
```

执行中发现 CHANGE_IMPACT 漏评估（如 QA 发现改动波及了未列出的模块）→ 打回 `software-architect` 补全 CHANGE_IMPACT.md（必要时补契约）→ 再补跑对应 Phase。

### I-4：回归验证

**调用 `testing-evidence-collector`**（迭代回归模式）

```
输入：
  - docs/iterations/ITER-{NNN}-{描述}/CHANGE_IMPACT.md（回归范围）
  - 本次 git diff（基线 commit..HEAD 或工作区）
  - 更新后的契约文件 + 变更记录

验证内容（全部要可复核证据，不依赖浏览器 / 截图）：
  A. 改动点验证：契约变更清单逐条核对实现（路径 / 字段 / 类型 / 错误码 / 迁移）
  B. 原有关键路径回归：CHANGE_IMPACT 列出的 P0 功能接口逐个做契约测试，
     完整跑一遍类型检查 + 已有单元测试 / 接口测试，确认无新增失败
  C. 残留扫描：被删除 / 重命名的旧字段名、旧接口路径在前后端代码中 grep 结果为 0
  D. 契约一致性：未改动的接口抽查实现仍与契约一致；契约版本号与变更记录已更新
  E. bug 修复类：先给出修复前的复现证据（或日志 / 失败用例），再给出修复后的通过证据

产出：
  - PASS 或 FAIL + 可复核证据（返回主会话）
  - 追加写入 docs/QA_REPORT.md，章节标题"## ITER-{NNN} 回归验证"

决策：
  PASS → 进入 I-5
  FAIL（重试 < 3）→ 按问题归属打回对应 agent（实现问题 → 实现 agent；契约 / 评估遗漏 → software-architect）
  FAIL（重试 >= 3）→ 暂停，向用户报告卡点
```

> I-3 中每个任务仍各自跑任务级 Dev-QA Loop；I-4 是**整次迭代**的回归关，两者不互相替代。

### I-5：文档更新 + 汇报

**调用 `technical-writer`**（增量更新模式）

```
输入：CHANGE_IMPACT.md + 契约变更记录 + 本次 git diff + QA_REPORT 的 ITER 章节
产出：
  - docs/CHANGELOG.md 追加一条（ITER 编号、日期、变更摘要、契约版本变化、是否破坏性、升级 / 迁移注意事项）
  - README.md 增量更新：仅当功能列表、环境变量、部署步骤、迁移说明有变化时改对应章节，不整篇重写
  - docs/API_DOC.md：API_CONTRACT 有变化时同步更新对应接口
```

主会话最后向用户汇报（格式见下）。

### Iteration 汇报格式

```
🔁 迭代完成：ITER-{NNN} {简短描述}【完整流程 / 快速通道】

📝 变更类型：新功能 / 优化 / bug 修复
📜 契约变更：PRD v1.2→v1.3 / API_CONTRACT v1.4→v1.5 / DB_SCHEMA 不变
💥 破坏性变更：否（或：是，已于 I-1 后经用户确认，迁移方案见 CHANGELOG）
🔄 重跑 Phase：3、5（TASK-B13、B07 重开）、6（TASK-F09）、7、8
⏭️  跳过 Phase：4（DB 不变）、9（部署配置不变）、10（非破坏性且无新功能）
🧪 回归：改动点 [n] 项 + 原有关键路径 [n] 条，testing-evidence-collector PASS
📁 文档：CHANGELOG.md 已追加，README / API_DOC [已更新 / 无需更新]

⚠️  遗留项：[若有]
```

### Iteration 反模式

- ❌ 跳过 I-1 / I-2 直接让实现 agent 改代码 → 契约与代码脱节，正是本模式要解决的问题
- ❌ 修改已执行过的迁移文件 → 线上库与迁移记录不一致
- ❌ 跳过的 Phase 不写理由 → 无法事后追溯"为什么没跑安全审查"
- ❌ 回归只验改动点、不跑原有关键路径 → 修好一个、坏掉两个
- ❌ 破坏性变更未经用户确认就落契约

---

## ► Phase 0：初始化目录

```bash
mkdir -p docs project-tasks
```

创建以下契约文件占位：
```
docs/
  PRD.md
  TECH_SPEC.md
  API_CONTRACT.md      ← 核心契约，前后端共同遵守
  DB_SCHEMA.md
  DESIGN_SYSTEM.md     ← UI 设计规范，frontend-developer 必须遵守
  DYNAMIC_CONTENT_MAP.md
  BACKEND_STATUS.md
  SECURITY_REPORT.md       ← Phase 7 安全审查报告
  REVIEW_REPORT.md         ← Phase 8 代码评审报告
  QA_REPORT.md             ← Phase 5/6 任务级 Dev-QA Loop 累计记录
  ACCEPTANCE_REPORT.md     ← Phase 10 最终验收报告
project-tasks/
  backend-tasklist.md
  frontend-tasklist.md
```

---

## ► Phase 1：需求分析

**调用 `product-manager`**

```
输入：用户原始需求
产出：docs/PRD.md

要求：
- 功能编号 F01/F02...（后续 agent 引用）
- 用户故事 Who/What/Why 格式
- 明确 MVP 范围（✅本期 / ❌不做）
- 非功能性需求（性能、安全、兼容性）

完成标志：docs/PRD.md 存在
```

**⏸ 人工检查点**：展示 PRD 功能列表，等待用户输入"继续"。

---

## ► Phase 2：技术架构 + 契约生成

**调用 `software-architect`**（这是整个流程最关键的阶段）

```
输入：读取 docs/PRD.md
产出：
  - docs/TECH_SPEC.md      技术栈、目录结构、环境变量、编码规范
  - docs/API_CONTRACT.md   ← 所有接口的完整定义（路径/方法/字段/错误码）
  - docs/DB_SCHEMA.md      所有表结构、字段类型、索引、外键关系

关键要求：
- API_CONTRACT 中每个接口必须包含：
    完整 Request body（字段名+类型+是否必填）
    完整 Response 200（字段名+类型+结构）
    所有错误码（HTTP状态码+error字段+触发条件）
- DB_SCHEMA 中每个字段必须标注类型、约束、索引原因
- 不得出现"等字段"、"其他参数"等模糊表述
- TECH_SPEC.md 必须包含"部署路径规范"章节，明确：
    APP_PATH（URL 前缀），VITE_API_BASE 的生产值和本地值
    前端 API 调用层规范（禁止硬编码 /api/...）
    .env / .env.production 的配置模板

完成标志：三个文件存在且无模糊表述，且 TECH_SPEC 包含部署路径规范
```

**⏸ 人工检查点**：展示 API 接口列表和表结构摘要，等待用户输入"继续"。

---

## ► Phase 2.5：UI 设计规范生成

**调用 `ui-designer`**

```
输入：
  - 读取 docs/PRD.md（了解产品定位和目标用户）
  - 读取 docs/TECH_SPEC.md（获取前端技术栈和适配方案）

产出:
  - docs/DESIGN_SYSTEM.md     颜色/字体/间距/圆角/阴影/组件规范
  - src/styles/variables.css  可直接引入的 CSS 变量文件

要求：
  - 颜色体系：品牌色 + 功能色 + 中性色，变量命名统一
  - 字体体系：基于实际适配方案（vw/rem）的字号梯度，最小不低于 16px
  - 间距体系：基于 4px 基础单位，覆盖页面边距/卡片内距/列表项高度
  - 组件规范：针对 PRD 中的核心组件（如：待办卡片/表单/按钮/空状态）
  - 移动端必须包含：安全区适配、可点击区域最小 44px、暗色模式变量

完成标志：docs/DESIGN_SYSTEM.md 和 src/styles/variables.css 均存在
```

---

## ► Phase 3：任务拆解

**由主会话亲自执行**（不调用子 agent）

读取 docs/API_CONTRACT.md 和 docs/DB_SCHEMA.md，生成：

**project-tasks/backend-tasklist.md**
```markdown
# 后端任务清单
> 基于 API_CONTRACT v1.0，每任务对应一个接口

### [ ] TASK-B01：实现 POST /api/v1/auth/login
- 对应契约：API_CONTRACT.md #auth-login
- 验收标准：返回 { token, user } 结构与契约一致

### [ ] TASK-B02：实现 GET /api/v1/users/:id
...
```

**project-tasks/frontend-tasklist.md**
```markdown
# 前端任务清单
> 每任务对应一个页面或核心组件

### [ ] TASK-F01：登录页面
- 调用契约：POST /api/v1/auth/login
- 验收标准：表单字段名与契约 Request 一致

### [ ] TASK-F02：用户详情页
...
```

---

## ► Phase 4：数据库实现

**调用 `database-optimizer`**

```
输入：读取 docs/DB_SCHEMA.md、docs/TECH_SPEC.md
产出：migrations/ 目录、model 文件、迁移运行器脚本、启动脚本

要求：
- 字段名严格与 DB_SCHEMA 一致，不得自行修改
- 若发现 Schema 有问题，写入 docs/DB_ISSUES.md 并停止
- 必须创建迁移运行基础设施（migrate.js/alembic stamp + start.sh）
  init.sql/init_db.py 只执行一次，生产环境表结构变更必须有独立迁移机制
- 不需要 Dev-QA Loop（纯结构性任务，无 UI 需要验证）

完成标志：
- migrations/ 存在，字段与 Schema 一致
- 迁移运行器脚本存在（scripts/migrate.js 或 alembic）
- 启动脚本存在（scripts/start.sh 或等价物）
```

若 docs/DB_ISSUES.md 存在 → 打回 `software-architect` 修正 DB_SCHEMA → 重试。

---

## ► Phase 5：后端实现（含任务级 Dev-QA Loop）

**逐任务执行以下循环：**

```
FOR 每个 project-tasks/backend-tasklist.md 中的 [ ] 任务：

  STEP 1 - 调用 backend-architect 实现该任务：
    输入：
      - 读取 docs/API_CONTRACT.md（必须第一步）
      - 读取 docs/DB_SCHEMA.md
      - 读取当前任务描述
    要求：
      - 严格按契约实现，路径/方法/字段名不得偏差
      - 若契约有歧义，写入 docs/BACKEND_STATUS.md 的 ISSUES 章节
    产出：该接口的实现代码

  STEP 2 - 调用 testing-evidence-collector 验证：
    输入：
      - 读取 docs/API_CONTRACT.md 中该接口定义
      - 扫描刚实现的代码文件
    验证内容：
      - 路径是否与契约一致
      - 返回字段名是否与契约一致
      - 错误处理是否覆盖契约中定义的状态码
    产出：
      - PASS 或 FAIL + 具体原因（返回主会话）
      - **追加写入 docs/QA_REPORT.md**（与 PRD/SECURITY/REVIEW 同级，累计记录所有任务级 QA 结果）

  STEP 3 - 决策：
    PASS → 将任务标记为 [x]，进入下一任务
    FAIL（重试 < 3）→ 将 QA 反馈传给 backend-architect，重新实现
    FAIL（重试 >= 3）→ 暂停，向用户报告卡点，等待介入

ALL 任务 PASS 后：
  检查 BACKEND_STATUS.md 的 ISSUES 章节
  若有未解决问题 → 打回 software-architect 更新契约 → 重跑受影响任务
```

---

## ► Phase 6：前端实现（含任务级 Dev-QA Loop）

**逐任务执行以下循环：**

```
FOR 每个 project-tasks/frontend-tasklist.md 中的 [ ] 任务：

  STEP 1 - 调用 frontend-developer 实现该任务：
    输入：
      - 读取 docs/API_CONTRACT.md（必须第一步）
      - 读取 docs/DESIGN_SYSTEM.md（必须第二步，所有样式数值来源）
      - 读取 docs/DYNAMIC_CONTENT_MAP.md（动态内容绑定规则）
      - 读取 docs/TECH_SPEC.md
      - 读取 docs/PRD.md（用户故事和UI需求）
      - 读取当前任务描述
    要求：
      - 所有 API 调用路径、字段名与契约完全一致
      - 所有颜色/字体/间距必须使用 DESIGN_SYSTEM 中定义的 CSS 变量
      - 不得硬编码任何颜色值、字号、间距值
      - 不得猜测接口结构

  STEP 2 - 调用 testing-evidence-collector 验证：
    验证内容（静态核查，不依赖浏览器/截图）：
      - UI 样式是否生效：CSS 静态核查（@media grep + viewport meta + 前端 typecheck）
      - API 调用字段名是否与契约一致
      - 表单提交和响应处理是否正确
      - 颜色/字体/间距是否使用了 CSS 变量（不得出现硬编码数值）
    产出：
      - PASS 或 FAIL + 可复核证据（typecheck/grep/单元测试输出，返回主会话）
      - **追加写入 docs/QA_REPORT.md**（与 PRD/SECURITY/REVIEW 同级，累计记录所有任务级 QA 结果）

  STEP 3 - 决策（同后端 Loop 规则）
```

---

## ► Phase 7：安全审查

**调用 `security-engineer`**

```
输入：扫描 src/ 目录
产出：docs/SECURITY_REPORT.md

重点检查：
- SQL 注入、XSS、CSRF
- 接口鉴权是否缺失
- 硬编码密码/密钥
- 文件上传未校验

完成标志：SECURITY_REPORT.md 存在
若发现高危问题 → 打回对应 agent 修复 → 重新扫描
```

---

## ► Phase 8：代码 Review

**调用 `code-reviewer`**

```
输入：git diff（或全量代码），读取 docs/TECH_SPEC.md（规范参考）
产出：docs/REVIEW_REPORT.md

检查项：代码规范、性能问题、可维护性、错误处理
若有 MUST FIX 级别问题 → 打回对应 agent → 重新 review
```

---

## ► Phase 9：DevOps 配置

**调用 `devops-automator`**

```
输入：读取 docs/TECH_SPEC.md（技术栈、环境变量、部署路径规范）
产出：Dockerfile、docker-compose.yml、CI/CD 配置

⚠️ 部署路径前缀检查（必须验证，不得跳过）：
  1. 确认 frontend/.env.production 存在且包含 VITE_API_BASE=/{APP_PATH}
  2. 确认 frontend/.env.production 包含 VITE_BASE_URL=/{APP_PATH}/
  3. 确认 vite.config.ts 中 base 使用环境变量（非硬编码）
  4. grep 扫描 frontend/src/ 确认无硬编码 /api/ 调用路径
  → 以上任意一项不满足，停止并报告问题，不得生成部署配置

完成标志：
  - 项目可通过 docker-compose up 启动
  - 部署路径前缀检查全部通过
```

---

## ► Phase 10：最终验收

**调用 `reality-checker`**

```
输入：
  - 读取 docs/API_CONTRACT.md
  - 读取 project-tasks/ 所有任务清单（验证全部 [x]）
  - 读取 docs/SECURITY_REPORT.md
  - 读取 docs/REVIEW_REPORT.md
  - 读取 docs/QA_REPORT.md（累计 QA 记录）
  - 运行完整用户旅程测试（端到端/接口契约测试证据，不依赖截图）

判决规则：
  - 默认判决：NEEDS WORK（必须有压倒性证据才能 READY）
  - READY 条件：
      ✅ 所有任务清单项均为 [x]
      ✅ 无未解决的安全高危问题
      ✅ 用户核心流程有端到端/接口测试通过记录
      ✅ API_CONTRACT 中所有接口均有测试通过记录

产出：
  - 判决摘要（返回主会话）
  - **写入 docs/ACCEPTANCE_REPORT.md**（与 PRD/SECURITY/REVIEW/QA 同级，覆盖式，每次验收刷新）

完成标志：
  - reality-checker 输出 READY
  - docs/ACCEPTANCE_REPORT.md 存在且判决为 READY
```

---

## ► Phase 11：文档

**调用 `technical-writer`**

```
输入：读取 docs/ 所有文件 + 项目源码结构
产出：
  - README.md（必须包含以下章节，缺一不可）：
      * 项目说明 + 技术栈
      * 本地开发启动方式
      * 环境变量说明表格
      * 首次部署步骤（创建目录、写 .env、配置 nginx location、触发 Actions）
      * 代码更新后再次部署流程
      * 数据库迁移使用说明（如何新增迁移文件、命名规范）
      * 项目目录结构
  - docs/API_DOC.md（基于 API_CONTRACT 的可读版文档）
```

---

# Subagent 中断与续传机制

## 背景

Subagent 执行长任务时可能撞到平台错误（502 upstream unreachable、context length exceeded、max turns 等），导致 agent 状态丢失并返回 error。但 agent 在死之前通过 Write/Edit 落盘的文件**仍保留在磁盘上**——Write 是原子的，文件要么完整要么不存在。

默认情况下，重新派遣同一类 subagent 不会"知道"这些残留文件，会按 prompt 从零开始，可能覆盖之前的工作。本机制要求主会话在派遣长任务前明示 deliverables，遇到中断时先做差异比对再续传。

## 触发条件

Subagent 返回的不是"任务完成"，而是以下任一：
- 平台错误：`API Error: 502`、`upstream unreachable`、`network` 等
- 资源耗尽：`context length exceeded`、`max turns exceeded`、`prompt is too long`
- Agent 中途汇报："我需要确认"、"遇到 X 问题请指示"

⚠️ 这与下方"打回重试总规则"区分——后者处理"agent 完成但 QA FAIL"，本机制处理"agent 根本没完成"。

## 派遣长任务前的标准动作（事前）

派 Phase 4 / 5 / 6 / 7 / 10 这种长任务 agent 时（Iteration 模式下的 I-1 影响评估、I-3 重跑的这些 Phase 同理），prompt 必须包含一段 **deliverables 清单**：

```
预期产出文件（如已部分存在，是上次中断遗留，请勿覆盖；只补齐缺失项）：
  - backend/src/app.js
  - backend/src/routes/auth.js
  - backend/src/controllers/auth.js
  - ...
```

这份清单主会话自己也要记下来（写进 task 描述或临时变量），中断时要用。

## Subagent 报错后的处理（事中）

按以下顺序，**不要跳步**：

1. **不要立即重派**
2. **`ls` 一遍 deliverables 清单中每个路径**，分类：
   - ✅ 已存在 = 上次中断前完成
   - ❌ 不存在 = 缺失
3. **抽查 1–2 个关键已存在文件**（如入口、controller），Read 看一眼确认不是空文件或半截内容。Write 工具是原子的，正常情况文件都是完整的；但若 agent 死在 Edit 工具中间，被 Edit 的文件可能不完整，要警惕。
4. **派一个同类型新 agent**，prompt 必须包含续传说明：
   ```
   ⚠️ 这是续传任务
   上次同任务执行时在第 N 个工具调用后中断（原因：{原始 error}）。
   以下文件已写完并经过验证，请勿重写、勿修改、勿删除：
     - {file1}
     - {file2}
     - ...
   只需完成以下缺失项：
     - {missing1}
     - {missing2}
   实施前可 Read 上面已存在文件了解上下文，但绝对不要 Write/Edit 它们。
   完成后简短汇报本次新写入的文件列表。
   ```

## 续传次数上限

| 中断类型 | 最大续传次数 | 超限处理 |
|---------|------------|---------|
| 平台错误（502 / network） | 2 次 | 暂停 → 向用户报告 |
| Context/turns 耗尽 | 1 次续传，第 2 次必须降级（拆任务或换 sonnet/haiku） | 暂停 |
| Agent 主动中途汇报 | 不走续传，按下方"打回重试"规则改 prompt 后重派 | — |

## 反模式

- ❌ Agent 报错后立即原样重派 → agent 不知道有残留文件，可能覆盖
- ❌ `rm -rf` 残留目录从零开始 → 浪费已完成的工作
- ❌ Prompt 里只写"继续上次任务" → agent 没有上下文，仍是从零

---

# 打回重试总规则

> 本表处理 **agent 完成了但产出有问题**（QA FAIL、安全问题、review 不过等）。
> 若 agent 因平台错误 / 上游不可达 / 资源耗尽**根本没完成**，先走上方 §Subagent 中断与续传机制，不要走本表。

| 触发条件 | 打回目标 | 最大重试 |
|---------|---------|---------|
| DB_ISSUES.md 存在 | software-architect | 2次 |
| BACKEND_STATUS.md 有未解决 ISSUES | software-architect → backend-architect | 2次 |
| 任务级 QA FAIL（样式硬编码）| frontend-developer | 3次/任务 |
| **任务级 QA FAIL（API 路径硬编码，未使用 VITE_API_BASE）** | **frontend-developer（零容忍，必须修复）** | **3次/任务** |
| 任务级 QA FAIL（接口问题）| 对应实现 agent | 3次/任务 |
| **Phase 9 部署路径前缀检查 FAIL** | **frontend-developer → devops-automator 重新验证** | **2次** |
| 安全高危问题 | 对应实现 agent | 2次 |
| REVIEW MUST FIX | 对应实现 agent | 2次 |
| reality-checker NEEDS WORK | 对应 agent | 1次 |
| Iteration：CHANGE_IMPACT 漏评估（实现 / QA 中发现未列出的受影响模块） | software-architect 补评估（必要时补契约）→ 补跑对应 Phase | 2次 |
| Iteration：I-4 回归 FAIL | 按问题归属打回实现 agent 或 software-architect | 3次 |
| 任何重试超限 | 暂停 → 向用户报告卡点 | — |

---

# 人工介入检查点

以下节点完成后主动暂停，展示摘要等待用户"继续"：

1. **Phase 1 后**：展示 PRD 功能列表（F01/F02...）
2. **Phase 2 后**：展示 API 接口列表 + 数据库表结构
3. **任意重试超限时**：展示失败详情，等待用户决策
4. **Iteration 模式 I-1 后（仅破坏性变更 / 建议改走全流程时）**：展示契约变更清单 + 风险与回滚方案

其余阶段自动执行，不打扰用户。

---

# 最终汇报格式

```
✅ 项目构建完成

📋 需求：[PRD 功能数量] 个功能，MVP 全部实现
🎨 设计：DESIGN_SYSTEM.md 已生成，[颜色/字体/间距] 规范已落地
🔌 接口：[已实现] / [契约定义总数] 个，全部通过 QA
🗄️  数据库：[表数量] 张表
🔒 安全：[高危/中危/低危问题数]，高危问题已全部修复
🧪 QA：所有任务通过 testing-evidence-collector 验证
✅ 验收：reality-checker 判决 READY
📁 文档：README.md + API_DOC.md 已生成

⚠️  遗留项：[若有跳过或降级处理的问题]
```
