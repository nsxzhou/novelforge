# NovelForge 项目开发 Review（基于《开发优先级.md》）

## 1. 评审范围与方法

- 评审范围：父仓库（文档与子模块治理）+ `backend` + `frontend`。
- 评审基线：以当前工作区代码与文档状态为准。
- 核验方式：
  - 文档口径一致性核对（`开发优先级.md`、父仓 `README.md`、后端 `README.md`）。
  - 对 `开发优先级.md` 的 1-8 项逐条做代码证据核验（路由、usecase、存储、迁移、测试）。
  - 运行基线验证：`cd backend && go test ./...`（已通过）。

## 2. 执行摘要

- 结论：后端 V1 主链路能力基本落地，`开发优先级.md` 的大部分“已完成”声明在代码中有证据支撑。
- 主要风险不在“后端是否有能力”，而在“全仓可交付性与一致性”：
  - 前端仍为占位，项目级最小闭环在全仓维度不可用。
  - 文档存在自相矛盾描述，会误导排期与验收。
  - 部分关键链路仍有并发与可运维性风险（对话确认并发写覆盖、迁移依赖人工步骤、成本指标口径未打通）。

## 3. Findings（按严重级别）

### 3.1 Critical

#### C-01 全仓闭环不可交付：前端仍为占位

- 问题：父仓定义为“前后端子模块项目”，但前端当前仅占位，缺少任何可用产品 UI/流程实现。
- 影响：在“整个项目”维度无法完成用户可用闭环，V1 只能算后端 API 就绪，不具备产品交付态。
- 证据：
  - [frontend/README.md](/Users/zhouzirui/code/AI/NovelForge/frontend/README.md#L1)
  - [README.md](/Users/zhouzirui/code/AI/NovelForge/README.md#L25)
- 建议修复：
  - 明确将当前阶段定义为“后端 V1”而非“项目 V1”。
  - 新建前端最小闭环里程碑（至少覆盖项目创建、资产编辑、章节生成、确认当前稿）。

### 3.2 High

#### H-03 缺失 CI/自动化质量闸门

- 问题：仓库未见 CI 工作流（如 `.github/workflows`），当前质量依赖人工执行测试。
- 影响：回归风险高，跨人协作时质量不稳定；子模块升级可能引入静默破坏。
- 证据：
  - 代码库未发现 CI 配置（本地检索结果为空）。
- 建议修复：
  - 增加最小 CI：`go test ./...`、`go vet`、`golangci-lint`（可分阶段）。
  - 父仓增加子模块变更检查规则（至少校验 `backend` 基线测试）。

### 3.3 Medium

#### M-03 指标 `token_usage` 仍是固定 0，成本可观测性未闭环

- 问题：章节与对话埋点虽有 `token_usage` 字段，但生成成功/失败都写入 0。
- 影响：无法支持成本分析、模型对比和预算治理，V1.1 看板价值受限。
- 证据：
  - [backend/internal/service/chapter/usecase.go](/Users/zhouzirui/code/AI/NovelForge/backend/internal/service/chapter/usecase.go#L633)
  - [backend/internal/service/conversation/usecase.go](/Users/zhouzirui/code/AI/NovelForge/backend/internal/service/conversation/usecase.go#L558)
- 建议修复：
  - 从 LLM 响应接入真实 usage（prompt/completion/total token）并回填 `GenerationRecord` 与 metric。

## 4. 《开发优先级.md》1-8 项对照结论

| 项 | 文档结论 | 代码核验 | 审查结论 | 注记 |
|---|---|---|---|---|
| 1 领域模型与存储边界 | 已完成 | 有 | 成立 | 无阻断 |
| 2 项目/资产 API | 已完成 | 有 | 成立 | 无阻断 |
| 3 LLM 客户端 + Prompt | 已完成 | 有 | 基本成立 | `asset_generation` 配置-能力映射不清 |
| 4 对话微调 | 已完成 | 有 | 基本成立 | Confirm 并发一致性风险 |
| 5 章节生成主链路 | 已完成 | 有 | 成立 | rewrite 参数处理一致性待修 |
| 6 当前稿确认 | 已完成 | 有 | 成立 | 与 conversation confirm 的并发策略不一致 |
| 7 埋点与可观测性 | 已完成 | 有 | 部分成立 | token 成本口径未打通 |
| 8 测试与回归 | 已完成 | 有 | 基本成立 | 缺 CI 自动闸门 |
