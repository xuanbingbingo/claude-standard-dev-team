<p align="center">
  <img src="assets/cover.png" alt="Claude Standard Dev Team" width="100%" />
</p>

# Claude Standard Dev Team

> 让 Claude Code 拥有一支 12 人 AI 软件开发团队 + 1 位总指挥，从需求到上线全流程自动跑通。

> ⚙️ **版本演进**：D1 视频录制时是 11 人版本 → 仓库主叙事迭代到 **12 人 + 1 总指挥**（当前默认）→ 进阶用户可启用基建层 3 个 agent 升级到 **16 人**完整部署闭环（见下方[进阶：完整部署链路](#进阶完整部署链路可选)）。这套团队仍在进化。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Compatible-blue.svg)](https://claude.com/claude-code)

---

## 这是什么

一套面向 [Claude Code](https://claude.com/claude-code) 的 **agent 团队配置**，把"软件开发"拆成 12 个专业岗位 + 1 位总指挥，按真实研发团队的协作链路串起来：

- **不再是 1 个 AI 一锅煮**：每个 agent 只干一件事，互不交叉
- **契约驱动**：先定 PRD/API/Schema，再让所有人照契约写
- **任务级 QA 闭环**：实现一个接口，立刻独立验证，FAIL 自动打回重做
- **零容忍硬编码**：API 路径硬编码必须打回——这是被部署翻车真实教训出来的

最适合用 Claude Code 做**中型应用**（前后端 + 数据库 + 部署）的人——不论你是独立开发者、一人公司还是小团队。

---

## 团队架构

```
                    ┌─────────────────────────┐
                    │     orchestrator        │
                    │   总指挥，不写代码        │
                    └────────────┬────────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
┌───────▼─────┐         ┌────────▼────────┐      ┌────────▼─────┐
│  规划层 2   │         │   实现层 5      │      │  质量层 4     │
├──────────── │         ├─────────────────│      ├──────────────│
│ pm          │         │ db-optimizer    │      │ ev-collector │
│ sw-architect│         │ backend-arch    │      │ sec-engineer │
│             │         │ ui-designer     │      │ code-reviewer│
│             │         │ frontend-dev    │      │ reality-chk  │
│             │         │ devops-automator│      └──────────────┘
└─────────────┘         └─────────────────┘
                                                  ┌──────────────┐
                                                  │  文档层 1    │
                                                  │ tech-writer  │
                                                  └──────────────┘
```

**总指挥 1 + 团队成员 12 = 12 人 AI 团队（不含总指挥）**

| 层级 | Agent | 职责 |
|---|---|---|
| **总指挥** | orchestrator | 不写代码，只调度其他 agent，把控 11 个阶段（CEO 角色，不算"团队成员"） |
| **规划** | product-manager | 把模糊需求拆成结构化 PRD + 用户故事 + 验收标准 |
| **规划** | software-architect | 技术选型 + 生成 API_CONTRACT / DB_SCHEMA / TECH_SPEC（最关键） |
| **实现** | ui-designer | 视觉规范 / 设计系统 / variables.css |
| **实现** | database-optimizer | 数据库迁移文件 + Model 层 + 索引设计 |
| **实现** | backend-architect | 严格按 API 契约实现接口 |
| **实现** | frontend-developer | 前端实现，禁止硬编码 API 路径 |
| **实现** | devops-automator | Dockerfile / docker-compose / CI/CD / 部署路径前缀检查 |
| **质量** | testing-evidence-collector | 任务级 QA，独立验证每个接口字段路径 |
| **质量** | security-engineer | 威胁建模 + 漏洞扫描 + OWASP 检查 |
| **质量** | code-reviewer | 正确性 / 可维护性 / 性能复审 |
| **质量** | reality-checker | 最终验收官，默认"需要返工"，要压倒性证据才放行上线 |
| **文档** | technical-writer | README / API 文档 / 教程，让别人能读懂 |

---

## 工作流程（11 个阶段）

```
Phase 0   orchestrator 创建项目目录                       （10 秒）
Phase 1   → product-manager      → PRD.md
   ⏸ 人工检查点 #1：确认功能范围
Phase 2   → software-architect   → API_CONTRACT.md / DB_SCHEMA.md / TECH_SPEC.md
   ⏸ 人工检查点 #2：确认接口契约（最关键节点）
Phase 2.5 → ui-designer          → DESIGN_SYSTEM.md / variables.css
Phase 3   orchestrator 自己拆任务清单
Phase 4   → database-optimizer   → migrations/
Phase 5   → backend-architect    │ ┐ Dev-QA Loop
          → ev-collector         │ │ 逐任务循环
Phase 6   → frontend-developer   │ ┐ Dev-QA Loop
          → ev-collector         │ │ 逐任务循环
Phase 7   → security-engineer    → SECURITY_REPORT.md
Phase 8   → code-reviewer        → REVIEW_REPORT.md
Phase 9   → devops-automator     → Dockerfile + 部署
Phase 10  → reality-checker      → READY 或 NEEDS WORK
Phase 11  → technical-writer     → README + API_DOC
完工 ✅
```

> 详见 [WORKFLOW.md](WORKFLOW.md)：每个 Phase 在做什么、Dev-QA Loop 怎么转、打回机制怎么走。

---

## 进阶：完整部署链路（可选）

12 个核心 agent 跑完 Phase 11 你已经拿到**完整可上线的代码库**——但"如何把它真正部署到一台云服务器"还差最后一步。

仓库 `agents/` 下额外提供了 3 个**基建层 agent**，让你一键打通"从代码到上线"：

| Agent | 职责 |
|---|---|
| `infra-bootstrap-agent` | 服务器一次性初始化：Docker 网络 + 中央 Nginx 网关 + 共享 MySQL |
| `app-deploy-agent` | 按 `deploy.yaml` 把任意应用部署到已初始化的服务器 |
| `deploy-yaml-schema` | `deploy.yaml` 字段契约（schema 文档，不是 agent） |

**典型用法**（一台服务器跑多个应用，路径前缀路由）：

```
新服务器 → infra-bootstrap-agent（一次）
新应用   → 仓库根目录写 deploy.yaml → app-deploy-agent
```

⚠️ **注意**：基建层带有作者私有的部署约定（共享网关 / 共享 MySQL / `/opt/apps/{app}` 目录布局），适合**一人公司 / 独立开发者**多应用复用一台服务器的场景。如果你有不同的部署偏好（K8s / 多服务器 / 云原生托管），跳过基建层、保留 12 个核心 agent 即可。

---

## 5 分钟上手

### 1. 装入 Claude Code

```bash
git clone https://github.com/xuanbingbingo/claude-standard-dev-team.git
cp claude-standard-dev-team/agents/*.md ~/.claude/agents/
```

### 2. 在 Claude Code 里说一句话启动

```
使用标准团队帮我开发一个 todo app
```

orchestrator 会自动接管：扫描需求 → 调 product-manager 写 PRD → 给你看 PRD 让你确认 → 调 software-architect 写 API 契约 → ... → 一路跑到 Phase 11 出文档。

详细安装/卸载/排错见 [INSTALL.md](INSTALL.md)。

### 3. 看着它跑

整个过程**只在 Phase 1 / Phase 2 后两次暂停**让你确认（PRD 范围 + API 契约），其他全自动。一个中等复杂度的应用从需求到部署完整跑完大约 **30-90 分钟**（取决于规模），主对话 token 消耗很低。

---

## 适合谁用

✅ 你正在用 Claude Code 写**中型应用**（前后端 + DB + 部署）  
✅ 你想要**契约驱动 + QA 闭环**而不是 1 个 AI 一把梭  
✅ 你被 AI 写出"看起来对、跑起来错"的代码坑过  
✅ 你需要**真实可上线**的产物，不只是 demo

❌ **不适合**：纯前端原型 / 单文件脚本 / 一次性小工具——杀鸡用牛刀  
❌ **不适合**：你还在评估 Claude Code 是否值得用——先用裸 Claude Code 跑通一遍再说

---

## 设计哲学（为什么这么编排）

| 设计原则 | 体现在哪 |
|---|---|
| **契约第一** | Phase 2 是最关键节点，后面 9 个 Phase 全都依赖契约 |
| **评审者 ≠ 实现者** | ev-collector 独立验证 backend / frontend 的产出，不让一个 agent 自己写自己测 |
| **打回有上限** | 重试 3 次还不行就暂停问用户，不死循环 |
| **人工介入点要少** | 只在 Phase 1/2 暂停，避免每步都打扰 |
| **职责单一** | 每个 agent 只干一类事 |
| **质量 > 速度** | 宁可串行也不并行（前后端不能同时改代码） |
| **零容忍硬编码** | API 路径硬编码必须打回（来自部署翻车的真实教训） |

---

## 与"裸 Claude Code"的区别

| 维度 | 裸 Claude Code | 标准团队 |
|---|---|---|
| 角色边界 | 1 个 AI 全干 | 12 个 agent 各司其职 |
| 契约约束 | 无（边写边改） | 必须先定 PRD/API/Schema |
| QA 验证 | 写完了说"应该 OK"| 任务级 QA Agent 独立验证 |
| 打回机制 | 没有 | 4 类规则 + 重试上限 |
| 部署路径检查 | 容易硬编码翻车 | Phase 9 强制检查 basePath |
| 适用规模 | 单文件 / 小脚本 | 中型应用 |

---

## 致谢

这套配置来自我经营**玉枢云创科技**一人公司过程中真实踩过的坑：

- 被 AI 改了字段名前端不知道翻过车
- 被部署 basePath 硬编码翻过车
- 被"AI 说写完了实际没写完"翻过车
- 被"AI 自己写自己测说 OK"翻过车

每一条零容忍规则背后都是一次真实事故。希望分享出来能让更多独立开发者少踩这些坑。

---

## License

[MIT](LICENSE) © 2026 斌哥-一人公司

如果这套配置帮到你，欢迎在 GitHub 上给个 ⭐ —— 让更多 Claude Code 用户看见。

---

## 相关链接

- [Claude Code 官方](https://claude.com/claude-code)
- [Claude Code agent 文档](https://docs.claude.com/en/docs/claude-code/sub-agents)
