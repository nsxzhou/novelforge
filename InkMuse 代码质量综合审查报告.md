# InkMuse 代码质量分阶段修复计划

> 本文档基于现有代码质量审查结果整理而成，目标是把原先按审查维度展开的 finding 列表，重组为可执行的分阶段修复路线。

## 导语

InkMuse 当前仓库采用父仓库 + 子模块协作方式，`frontend/` 与 `backend/` 通过 Git submodule 管理。父仓库主要承载总览文档、运行说明、跨仓约定与子模块指针，实际业务整改将分别落在前端和后端子仓库内推进。

本计划遵循“前端先、后端后、跨模块收口”的顺序：先处理用户可感知最强、同时阻塞后续演进的页面编排和流式性能问题，再进入后端核心用例与数据链路的结构修复，最后统一收口共享契约、质量门和测试基线。

## 修复原则

- 模块分治：阶段划分对齐真实仓库边界，不混淆父仓库、前端子模块、后端子模块职责。
- 前端优先：优先处理工作台、写作页、流式渲染和懒加载等直接影响使用体验的问题。
- 最小改动面：每一阶段都以收敛当前主问题为目标，不把 unrelated refactor 混入同一轮整改。
- 单一真相源：共享 DTO、枚举和 schema 继续遵循“后端稳定定义 -> `contractgen` -> 前端 generated contracts”路径，不在前端重复造协议。
- 排序稳定：问题清单按“高风险或 P0 在前，中风险或 P1 其次，低风险与 P2-P3 收尾”排序；同档位保持与原报告一致的出现顺序。

## 总览 / 基线

- 当前规范一致性评分基线：`68/100`
- 当前问题总量：`62` 条 finding
- 阶段顺序：`Phase 1 前端核心页面与流式性能 -> Phase 2 前端共享层与规范治理 -> Phase 3 后端核心用例与数据链路 -> Phase 4 后端装配层与基础设施治理 -> Phase 5 跨模块契约、测试基线与最终收口`

| 阶段 | 范围 | 修复目标 | 预期产出 | 验收标准 |
|------|------|----------|----------|----------|
| Phase 1 | 前端核心页面、read model、controller、流式渲染 | 先压住工作台与写作主路径上的架构膨胀和性能主风险 | 页面壳与 tab 容器拆分方案、写作页子 hook 化、主路径懒加载与失效收敛 | 工作台与写作页主路径职责清晰，首屏与流式交互不再被过载容器拖累 |
| Phase 2 | 前端 shared/api、shared/lib、局部规范项 | 清理共享层重复实现和中低风险规范债务 | 通用 query helper、stream endpoint 工厂、低价值别名清理、局部规范修复 | 前端共享层不再重复造轮子，低成本问题一次性收束 |
| Phase 3 | 后端 bootstrap、handler、service、repository、SSE、提取链路 | 处理后端核心结构与高负载数据链路问题 | bootstrap 分层、handler 拆分、use case 去 God 化、列表降载、SSE 与提取链路瘦身 | 后端主用例分层回到既有架构，热点查询和流式链路具备可扩展性 |
| Phase 4 | 后端 routes、基础设施组织、命名和低成本清理 | 收敛装配层与基础设施侧的中低风险治理项 | registrar 化路由、接口缩口、空包装层与死代码清理、补齐基础 GoDoc | 装配入口与基础设施命名/职责更稳定，可维护性提升 |
| Phase 5 | 共享契约、质量门、E2E 基线、最终收口 | 在结构调整稳定后统一关闭跨模块治理项 | generated contracts 回归单一真相源、zero-warning 质量门、E2E 去占位化、共享 DTO 文档补齐 | 前后端共享边界一致，质量门和测试基线可作为后续整改收口标准 |

## Phase 1｜前端核心页面与流式性能修复

这一阶段先处理工作台与写作页主路径，因为这部分同时承载路由编排、查询、控制器、编辑和流式交互，是当前最直接暴露给用户、也最容易拖慢后续整改节奏的区域。只有先把页面壳、tab 容器、read model 和 orchestration 主体拆开，后续 shared 层治理和后端接口收敛才不会继续被页面级 God Object 反复放大。

### 修复目标

- 降低 `project-workbench-page` 与 `write-page` 的页面级编排密度。
- 把 read model、controller、orchestration 从万能入口拆回场景化边界。
- 优先压住流式渲染风暴、主工作台全量预取和重复 refetch 等体验瓶颈。

### 完成产出

- 工作台页面已收敛为“路由页壳 + `features/projects/workbench` 容器”，tab 装配、分页资产加载、记忆层与知识图谱激活逻辑从页面文件下沉。
- 写作页已收敛为“页面壳 + 组合 hook + 分组 screen props”，自动保存、生成流、派生状态刷新、确认守卫与导航逻辑边界可独立测试。
- 主路径已切换到项目聚合计数、章节摘要列表和流式批量提交策略；记忆轮询改为可见性感知的退避策略。

### 验收结果

- 页面级组件已不再同时承担路由、查询、编辑、导出、tab 切换和多域 controller 调度。
- 主要流式交互链路已具备 `rAF/80ms` 批量提交策略，避免按 chunk 直接驱动高频重渲染。
- 工作台和写作页的查询装配边界已可被单独测试和独立演进。

### 完成状态

- 状态：`已完成（2026-03-31）`
- 后端验证：`go test ./...`、`go run ./cmd/contractgen`
- 前端验证：`npm run lint`、`npm run test`、`npm run build`
- 验证结果：后端全量测试通过；前端 `30` 个测试文件、`109` 个测试全部通过；前端构建通过。

### 纳入问题清单

#### 高风险 / P0

1. **来源：架构审查 1**  
   Finding: 工作台页面成为页面级编排中心（路由/查询/控制器/编辑/导出/切 tab 全部集中）  
   Location: `frontend/src/pages/project-workbench-page.tsx:202`  
   Severity: 高  
   Recommendation: 拆为 tab 容器 + 页面壳  
   Refactor Priority: P0
   完成情况：已将页面文件收敛为路由入口，核心编排下沉到 `features/projects/workbench/project-workbench.tsx`，tab 内容与页面壳状态不再挤在 `pages/` 层。
   验证方式：`npm run test` 覆盖 `src/features/projects/workbench/project-workbench.test.tsx`，并通过 `npm run build` 验证页面装配链路。

2. **来源：架构审查 3**  
   Finding: `useProjectWorkbenchData` 默认全量加载 `assets/chapters`，形成 mega read model  
   Location: `frontend/src/features/projects/authoring-read-models.ts:340`  
   Severity: 高  
   Recommendation: 按 tab 场景拆分 read model  
   Refactor Priority: P0
   完成情况：工作台主路径已改为项目聚合详情 + 大纲/角色轻量资源 + 章节摘要列表 + 资产分页加载；工作台容器不再依赖全量资产/整章正文 read model。
   验证方式：`src/features/projects/workbench/project-workbench.test.tsx` 校验概览页不触发 memory/KG 查询、资产 tab 按页加载；`src/features/projects/authoring-data.test.tsx` 校验章节摘要与写作页轻量数据流。

3. **来源：架构审查 5**  
   Finding: 写作页 `useWritePageOrchestration` 成为 God Hook  
   Location: `frontend/src/pages/write-page.hooks.ts:451`  
   Severity: 高  
   Recommendation: 拆查询/自动保存/生成流/确认守卫/导航子 hook  
   Refactor Priority: P0
   完成情况：写作页已保留组合 hook 入口，但自动保存、生成流、派生刷新、轮询策略和保存队列拆为独立子逻辑；页面文件仅保留路由和本地 UI 状态装配。
   验证方式：`src/pages/write-page.hooks.test.tsx`、`src/pages/write-page.test.tsx`，以及全量 `npm run test`。

4. **来源：性能优化审查 2**  
   Finding: 流式 chunk 每次 `setState` 导致渲染风暴  
   Location: `frontend/src/shared/lib/use-stream-task.ts:113`  
   Severity: 高  
   Recommendation: `rAF/50-100ms` 批量提交  
   Refactor Priority: P0
   完成情况：`use-stream-task` 已改为缓冲 chunk，并在下一帧或 `80ms` 定时器统一 flush，终态事件前强制冲刷缓冲内容。
   验证方式：`src/shared/lib/use-stream-task.test.ts` 覆盖批量提交、终态 flush、取消后忽略迟到回调等回归场景。

5. **来源：性能优化审查 3**  
   Finding: 工作台默认预取全量资产与章节  
   Location: `frontend/src/pages/project-workbench-page.tsx:224`  
   Severity: 高  
   Recommendation: 按 tab 懒加载  
   Refactor Priority: P0
   完成情况：工作台默认只加载项目聚合信息、概览必需资源和章节摘要；记忆层、知识图谱与分页资产查询均改为切 tab 后激活。
   验证方式：`src/features/projects/workbench/project-workbench.test.tsx` 校验 overview 首屏不拉 memory/KG 数据，切 tab 后才激活对应请求。

6. **来源：性能优化审查 4**  
   Finding: 资产/章节固定大分页（500）且非真分页  
   Location: `frontend/src/features/projects/authoring-read-models.ts:187`  
   Severity: 高  
   Recommendation: 后端分页 + 前端增量加载  
   Refactor Priority: P0
   完成情况：后端新增 `chapter-summaries` 轻量分页接口，前端 assets tab 改为 `50` 条一页的显式“加载更多”，章节树/写作页侧栏改为消费摘要列表而非整章正文。
   验证方式：后端 `go test ./...` 覆盖 handler/integration 回归；前端 `src/features/projects/workbench/project-workbench.test.tsx` 覆盖资产分页加载行为。

7. **来源：设计模式评估 1**  
   Finding: 页面级 God Object（Workbench）  
   Location: `frontend/src/pages/project-workbench-page.tsx:78`  
   Severity: 高  
   Pattern Concern: `SRP/God Object`  
   Recommendation: 控制器和 tab 容器拆分  
   Refactor Priority: P0
   完成情况：workbench 已拆为页面入口、feature 容器、分页资产状态、记忆控制器组合与知识图谱容器，原页面级 God Object 已解除。
   验证方式：`src/features/projects/workbench/project-workbench.test.tsx` 与全量 `npm run build`。

8. **来源：设计模式评估 3**  
   Finding: 写作页 orchestration 过厚  
   Location: `frontend/src/pages/write-page.hooks.ts:451`  
   Severity: 高  
   Pattern Concern: `God Hook`  
   Recommendation: orchestration 仅做组合  
   Refactor Priority: P0
   完成情况：写作页 orchestration 已退回组合层，screen 层参数收敛为 `frame / view / actions`，编辑页 header/actions/body 也已拆成独立子组件。
   验证方式：`src/pages/write-page.test.tsx`、`src/pages/write-page.hooks.test.tsx`、`npm run build`。

#### 中风险 / P1

1. **来源：架构审查 2**  
   Finding: `useProjectWorkbenchShell` 把业务控制逻辑放在页面文件  
   Location: `frontend/src/pages/project-workbench-page.tsx:78`  
   Severity: 中  
   Recommendation: 下沉到 `features/projects/workbench` 的专用 hooks  
   Refactor Priority: P1
   完成情况：shell 状态、导出、删除、编辑态和 tab URL 同步逻辑已从页面文件迁入 `features/projects/workbench` 容器。
   验证方式：`src/features/projects/workbench/project-workbench.test.tsx`、`npm run build`。

2. **来源：架构审查 4**  
   Finding: `useProjectMemoryController` 混合角色状态与时间线两个子域  
   Location: `frontend/src/features/projects/authoring-controllers.ts:136`  
   Severity: 中  
   Recommendation: 拆为 `useCharacterStateController / useTimelineController`  
   Refactor Priority: P1
   完成情况：记忆控制器已拆为 `useCharacterStateController` 与 `useTimelineController`，并在 MemoryPanel 中以组合 controller 方式消费。
   验证方式：`src/features/memory/memory-panel.test.tsx`、全量 `npm run test`。

3. **来源：性能优化审查 7**  
   Finding: 记忆控制器重复 invalidation 触发重复 refetch  
   Location: `frontend/src/features/projects/authoring-controllers.ts:157`  
   Severity: 中  
   Recommendation: 最小化失效集合  
   Refactor Priority: P1
   完成情况：角色状态 mutation 仅失效 latest + chapter scope；时间线 mutation 仅失效 timeline query，已移除额外的 `invalidateProjectMemory()` 级重复触发。
   验证方式：`src/pages/write-page.hooks.test.tsx` 中的派生刷新回归，以及全量 `npm run test`。

4. **来源：性能优化审查 8**  
   Finding: 派生状态 `1.5s` 轮询，异常时长期背景流量  
   Location: `frontend/src/pages/write-page.hooks.ts:371`  
   Severity: 中  
   Recommendation: 退避 + 最大时长 + 可见性降频  
   Refactor Priority: P1
   完成情况：已引入 `createDerivationRefetchInterval`，可见页前 `20s` 使用 `1.5s`，随后退避到 `5s`；隐藏页降为 `15s`；持续活跃超过 `3m` 自动停止。
   验证方式：`src/pages/write-page.hooks.test.tsx` 中新增轮询退避与隐藏页场景测试。

5. **来源：设计模式评估 2**  
   Finding: tab switch 直接感知多域 controller，呈现 Service Locator 倾向  
   Location: `frontend/src/pages/project-workbench-page.tsx:239`  
   Severity: 中  
   Pattern Concern: `Service Locator`  
   Recommendation: 每个 tab 独立容器  
   Refactor Priority: P1
   完成情况：workbench 内容已按 overview / assets / chapters / memory / knowledge-graph 分支容器化，tab 切换不再由页面壳直接操作多域 controller。
   验证方式：`src/features/projects/workbench/project-workbench.test.tsx` 检查 memory/KG 仅在对应 tab 激活后请求。

6. **来源：设计模式评估 4**  
   Finding: `useProjectResources` 配置对象膨胀  
   Location: `frontend/src/features/projects/authoring-read-models.ts:150`  
   Severity: 中  
   Pattern Concern: `Speculative Generality`  
   Recommendation: 场景化 hook 替代万能入口  
   Refactor Priority: P1
   完成情况：主工作台已切换到 feature 容器 + 分页资产状态 + 章节摘要接口组合；写作页与记忆面板均改为消费场景化的轻量资源，不再依赖 workbench 级万能数据入口。
   验证方式：`src/features/projects/authoring-data.test.tsx`、`src/features/projects/workbench/project-workbench.test.tsx`、全量 `npm run test`。

7. **来源：代码规范检查 2**  
   Finding: 页面容器过重  
   Location: `frontend/src/pages/project-workbench-page.tsx:202`  
   Severity: 中  
   Rule Violated: 入口收敛细节下沉  
   Recommendation: 拆 content 装配  
   Refactor Priority: P1
   完成情况：`pages/project-workbench-page.tsx` 已缩减为路由参数转发，真实 content 装配迁移到 `features/projects/workbench/project-workbench.tsx`。
   验证方式：`src/app/routes.test.tsx`、`src/features/projects/workbench/project-workbench.test.tsx`、`npm run build`。

8. **来源：代码规范检查 4**  
   Finding: `EditorScreenProps` 参数面过大  
   Location: `frontend/src/pages/write-page.screens.tsx:156`  
   Severity: 中  
   Rule Violated: 参数规模/可读性  
   Recommendation: 拆 view-model  
   Refactor Priority: P1
   完成情况：写作页编辑屏 props 已拆分为 `frame / view / actions` 三组，页面装配面和组件阅读成本显著收敛。
   验证方式：`src/pages/write-page.test.tsx`、`npm run lint`、`npm run build`。

9. **来源：代码规范检查 5**  
   Finding: `WriteEditorScreen` 职责过宽  
   Location: `frontend/src/pages/write-page.screens.tsx:179`  
   Severity: 中  
   Rule Violated: `SRP`  
   Recommendation: `header/actions/body` 子组件化  
   Refactor Priority: P1
   完成情况：`WriteEditorScreen` 已拆为 header、action bar 与 body 三个局部组件，screen 本身退回到视图拼装层。
   验证方式：`src/pages/write-page.test.tsx`、`npm run test`、`npm run build`。

## Phase 2｜前端共享层、冗余与规范治理

这一阶段在主页面结构稳定后推进，因为 shared 层抽象、query helper、stream 工厂和局部规范项都依赖于前一阶段的真实使用边界。如果先做 shared 层统一，后续页面拆分时仍会反复回滚接口与工具函数，收益会被稀释。

### 修复目标

- 清理前端 shared/api 与 shared/lib 中已经显性的重复实现。
- 收敛轻量性能问题和低成本规范问题，避免这些债务在主路径重构后继续扩散。
- 把前端共享层恢复为“复用能力集中、UI 页面少拼装”的稳定支点。

### 完成产出

- `shared/api` 已新增统一 `query-helpers`，`assets / projects / chapters` 的 query string 拼装已收敛到同一入口。
- `shared/api/sse-client` 已新增 `createStreamEndpoint` 工厂，`assets` 与 `chapters` 中标准 schema-based SSE 包装已统一复用。
- timeout policy 已改为客户端缓存并在 provider 新增、更新、删除后失效；低价值类型别名、未使用依赖、无理由 lint 压制和硬编码语义色值均已收口。

### 验收结果

- shared/api 中已不再跨文件重复拼装 query string，标准流式接口也不再散落重复 `streamRequestWithSchema(...)` 包装。
- SSE timeout policy 不再为每次流式请求额外请求一次 `/llm/timeout-policy`；KG 边过滤和未写章节推导均已改为 `Set` 级查找。
- 局部规范例外已完成处置：GhostText 存储读写具备最小类型约束，`useToast` lint 例外已补原因说明，写作页提示文案已切回语义 token。

### 完成状态

- 状态：`已完成（2026-03-31）`
- 前端验证：`npm run lint`、`npm run test`、`npm run build`
- 辅助核验：`xxd -g 1 -l 4 frontend/src/pages/project-workbench-page.tsx`
- 验证结果：前端 lint 通过；全量 `34` 个测试文件、`123` 个测试全部通过；前端构建通过；`project-workbench-page.tsx` 文件头已确认不存在 `BOM`。

### 纳入问题清单

#### 中风险 / P1

1. **来源：代码冗余检测 1**  
   Finding: 前端多个 API 文件重复手写 query string 组装  
   Location: `frontend/src/shared/api/assets.ts:29`  
   Severity: 中  
   Recommendation: 抽 `buildQueryString/listRequest`  
   Refactor Priority: P1
   完成情况：已新增 `frontend/src/shared/api/query-helpers.ts`，并将 `listAssets`、`listProjects`、`listChapters`、`listChapterSummaries` 的 query string 组装统一改为 `buildPathWithQuery(...)`。
   验证方式：`npm run test -- src/shared/api/assets.test.ts src/shared/api/chapters.test.ts src/shared/api/projects.test.ts`，以及全量 `npm run test`。

2. **来源：代码冗余检测 2**  
   Finding: 流式 API 包装函数同构重复  
   Location: `frontend/src/shared/api/chapters.ts:70`  
   Severity: 中  
   Recommendation: 抽 `createStreamEndpoint` 工厂  
   Refactor Priority: P1
   完成情况：已在 `shared/api/sse-client.ts` 中新增 `createStreamEndpoint`，并将 `assets` 与 `chapters` 中标准 schema-based stream wrapper 收敛到统一工厂；`brainstorm` 与 `polish` 仍保留定制事件解析路径。
   验证方式：`src/shared/api/assets.test.ts`、`src/shared/api/chapters.test.ts` 校验工厂接线；全量 `npm run test`。

3. **来源：性能优化审查 1**  
   Finding: 每次 SSE 先请求 timeout policy，固定额外 RTT  
   Location: `frontend/src/shared/api/sse-client.ts:53`  
   Severity: 中  
   Recommendation: 缓存 timeout policy + provider 更新时失效  
   Refactor Priority: P1
   完成情况：`getTimeoutPolicy()` 已改为缓存 Promise，并在 `addProvider / updateProvider / deleteProvider` 成功后清空缓存；SSE 与 provider test 共用同一缓存结果。
   验证方式：`src/shared/api/llm-providers.test.ts` 覆盖缓存命中与 provider 变更后失效场景；全量 `npm run test`。

4. **来源：性能优化审查 6**  
   Finding: KG 边过滤 `O(E×N)`  
   Location: `frontend/src/features/knowledge-graph/kg-graph.tsx:43`  
   Severity: 中  
   Recommendation: 预建可见节点 `Set/Map`  
   Refactor Priority: P1
   完成情况：KG 图谱数据构建已先生成可见节点 `Set`，边过滤不再对每条 edge 重复执行 `nodes.some(...)`。
   验证方式：`src/features/knowledge-graph/kg-graph.test.tsx` 与 `src/features/knowledge-graph/kg-panel.test.tsx`，以及全量 `npm run test`。

5. **来源：代码规范检查 7**  
   Finding: `eslint-disable any` 无理由注释  
   Location: `frontend/src/features/chapters/components/tiptap-editor.tsx:67`  
   Severity: 中  
   Rule Violated: `AGENTS 5.7`  
   Recommendation: 补最小类型或明确原因  
   Refactor Priority: P2
   完成情况：GhostText 扩展已补 `GhostTextStorageRecord` 与 `writeGhostTextStorage(...)`，`tiptap-editor` 不再使用 `any` cast 直接写 `editor.storage.ghostText`。
   验证方式：`npm run lint`、`npm run build`、全量 `npm run test`。

#### 低风险 / P2-P3

1. **来源：代码冗余检测 3**  
   Finding: `cancel/reset` 逻辑重复  
   Location: `frontend/src/shared/lib/use-stream-task.ts:80`  
   Severity: 低  
   Recommendation: 合并为内部 `clear()`  
   Refactor Priority: P2
   完成情况：`useStreamTask` 已将 `cancel/reset` 的重复清理路径收敛为内部共享 `clearState()`，对外行为保持不变。
   验证方式：`src/shared/lib/use-stream-task.test.ts` 与全量 `npm run test`。

2. **来源：代码冗余检测 4**  
   Finding: 无增益类型别名 `RewritePreviewResult`  
   Location: `frontend/src/shared/api/chapters.ts:36`  
   Severity: 低  
   Recommendation: 直接使用 `RewritePreviewResponse`  
   Refactor Priority: P3
   完成情况：`RewritePreviewResult` 已删除，消费方统一改为直接使用 generated contract 类型 `RewritePreviewResponse`。
   验证方式：`npm run lint`、`npm run build`、全量 `npm run test`。

3. **来源：代码冗余检测 5**  
   Finding: 无增益类型别名 `ApplyRefineInput`  
   Location: `frontend/src/shared/api/assets.ts:27`  
   Severity: 低  
   Recommendation: 统一为 `ApplyRefineAssetInput`  
   Refactor Priority: P3
   完成情况：`ApplyRefineInput` 已删除，资产控制器等消费方统一回到 `ApplyRefineAssetInput`。
   验证方式：`npm run lint`、`npm run build`、全量 `npm run test`。

4. **来源：代码冗余检测 7**  
   Finding: `@xyflow/react` 未使用  
   Location: `frontend/package.json:23`  
   Severity: 低  
   Recommendation: 移除依赖或注明保留原因  
   Refactor Priority: P3
   完成情况：`@xyflow/react` 已从 `frontend/package.json` 与 `package-lock.json` 中移除，连带无用的 `d3` 类型与 `zustand` 依赖树已清理。
   验证方式：`npm run build`、`npm run test`。

5. **来源：性能优化审查 5**  
   Finding: 未写章节计算 `O(规划×已写)`  
   Location: `frontend/src/features/projects/authoring-read-models.ts:272`  
   Severity: 中  
   Recommendation: 先构建 ordinal `Set`  
   Refactor Priority: P2
   完成情况：`useOutlinePlan()` 已改为先构建 `writtenOrdinals` 集合，再推导 `unwrittenPlannedChapters`，避免对每个 planned chapter 重复遍历已写章节列表。
   验证方式：`src/features/projects/authoring-data.test.tsx` 新增稀疏 ordinal 回归场景，并通过全量 `npm run test`。

6. **来源：代码规范检查 3**  
   Finding: 文件含 `BOM`  
   Location: `frontend/src/pages/project-workbench-page.tsx:1`  
   Severity: 低  
   Rule Violated: 格式一致性  
   Recommendation: 去 `BOM`  
   Refactor Priority: P3
   完成情况：当前 HEAD 中 `frontend/src/pages/project-workbench-page.tsx` 文件头已无 `BOM`，本阶段按“已满足”回写，不额外制造无效代码改动。
   验证方式：`xxd -g 1 -l 4 frontend/src/pages/project-workbench-page.tsx`。

7. **来源：代码规范检查 6**  
   Finding: 硬编码 slate 颜色绕过语义 token  
   Location: `frontend/src/pages/write-page.screens.tsx:321`  
   Severity: 低  
   Rule Violated: `AGENTS 5.6`  
   Recommendation: 用 `muted-foreground` 等语义 token  
   Refactor Priority: P3
   完成情况：写作页草稿提示文案的硬编码 `text-slate-300` 已替换为语义 token `text-muted-foreground`。
   验证方式：`npm run lint`、`npm run build`。

8. **来源：代码规范检查 8**  
   Finding: `useToast` lint 压制无明确理由  
   Location: `frontend/src/shared/ui/toast.tsx:20`  
   Severity: 低  
   Rule Violated: `AGENTS 5.7`  
   Recommendation: 写明原因或重构导出结构  
   Refactor Priority: P3
   完成情况：`useToast` 继续保留与 Provider 同文件导出，但已补充局部注释说明 react-refresh 例外原因，不再是无理由压制。
   验证方式：`npm run lint`、`npm run build`。

## Phase 3｜后端核心用例、Handler 与数据链路修复

这一阶段处理后端核心结构与热点数据链路。原因很直接：当前后端的 `bootstrap`、`chapter use case`、`handler` 和列表查询链路存在职责膨胀、跨层泄漏与 overfetch 问题，它们会直接限制前端懒加载、分页、流式稳定性和后续共享契约收口。只有先把核心用例和数据路径瘦身，跨模块治理才有稳定落点。

### 修复目标

- 把 `LoadBootstrap`、`ChapterHandler`、`ProjectHandler` 和 `chapter use case` 从 God 形态拆回既有分层。
- 处理列表 overfetch、DB 高频轮询、资产生成前全量加载和 SSE 全文聚合等主性能风险。
- 把服务层与 infra 之间的泄漏收敛到端口接口或薄适配器上。

### 预期产出

- 分层 bootstrap、按职责拆分的 chapter/project handler、场景化 chapter use case。
- 面向列表/详情分离的查询接口与存储层负载裁剪。
- 更轻量的提取、流式和资产生成链路，减少 DB 与内存热点。

### 验收标准

- handler 仅承担参数绑定、用例调用、响应映射，不再直接编排复杂并发/SSE 逻辑。
- service 层不再直接依赖 infra 类型，核心场景通过端口接口隔离。
- 热点查询与轮询链路不再以“全量加载 + 高频轮询 + handler 侧全文缓存”为默认策略。

### 完成产出

- `LoadBootstrap` 已拆为 `initStorage / initLLM / initUseCases / initHTTP` 四段初始化流程，`bootstrap.go` 不再承担全部装配细节。
- `ProjectHandler` 已拆为 CRUD 与 ideation/brainstorm 两组 handler，`ChapterHandler` 已拆为 query / write / stream / review 四组 handler，SSE 传输辅助逻辑改为复用 helper。
- chapter/asset 列表链路已切到真实 summary DTO：章节列表与 `chapter-summaries` 分离查询，资产列表改为摘要列表，前端改为按需 `getAsset()` 拉详情。
- `chapter` service 已引入本地 `llm / tx / extraction / kgSync / jsonExecutor` 端口接口，记忆新鲜度等待改为指数退避；`SSE` 全文累计已收回到 service-side stream execution。
- asset 生成/优化上下文已改走 metadata-first 摘要列表，extraction 的已知角色名加载已切到 title-only 查询；`service/project_scope.go` 已抽出公共 project/chapter scope 校验 helper。

### 验收结果

- handler 已回到薄层：chapter/project 的 HTTP 入口只做绑定、use case 调用、响应映射，brainstorm 与章节流式编排不再直接堆在单个 handler 体内。
- service 层已不再直接依赖 chapter 场景中的 concrete infra 类型；关键能力通过局部端口接口与通用 stream runner 隔离。
- 热点查询和流式链路已不再默认全量正文：章节 summary 查询不再回传正文，资产列表不再回传 `content`，SSE helper 不再在 transport 层再聚合一份全文，记忆等待改为有上限的指数退避。

### 完成状态

- 状态：`已完成（2026-03-31）`
- 后端验证：`go test ./...`、`go vet ./...`、`go run ./cmd/contractgen`
- 前端验证：`npm run lint`、`npm run test`、`npm run build`
- 验证结果：后端全量测试通过，`go vet` 通过，契约重新生成完成；前端 lint 通过，全量 `34` 个测试文件、`124` 个测试全部通过，前端构建通过。

### 纳入问题清单

#### 高风险 / P0

1. **来源：架构审查 6**  
   Finding: 启动装配 `LoadBootstrap` 过度集中  
   Location: `backend/internal/app/bootstrap.go:61`  
   Severity: 高  
   Recommendation: 分层 `initStorage/initLLM/initUseCases/initHTTP`  
   Refactor Priority: P0
   完成情况：`LoadBootstrap()` 已收敛为配置/迁移后调用 `initStorage()`、`initLLM()`、`initUseCases()`、`initHTTP()` 的分阶段组合根，`bootstrap.go` 不再内联全部装配过程。
   验证方式：`go test ./internal/app`、全量 `go test ./...`。

2. **来源：架构审查 8**  
   Finding: `ChapterHandler` 承担 CRUD + AI 同步 + SSE + DTO 转换  
   Location: `backend/internal/infra/http/handler/chapter.go:21`  
   Severity: 高  
   Recommendation: 拆 `CRUD/Generation/Review/Stream handlers`  
   Refactor Priority: P1
   完成情况：章节 HTTP 层已拆为 `chapter_query.go`、`chapter_write.go`、`chapter_stream.go`、`chapter_review.go` 四组处理器，主文件仅保留共享响应类型与 helper。
   验证方式：`go test ./internal/infra/http/handler`、全量 `go test ./...`。

3. **来源：架构审查 9**  
   Finding: `ProjectHandler` 混合 CRUD 与 brainstorming/guided AI  
   Location: `backend/internal/infra/http/handler/project.go:273`  
   Severity: 高  
   Recommendation: 拆 `ProjectIdeationHandler`  
   Refactor Priority: P1
   完成情况：项目 HTTP 层已拆为 `project_crud.go` 与 `project_ideation.go`，brainstorm SSE 编排进一步下沉到 `project_brainstorm_stream.go`。
   验证方式：`go test ./internal/infra/http/handler`、全量 `go test ./...`。

4. **来源：架构审查 10**  
   Finding: `chapter service` 对 infra 类型泄漏明显  
   Location: `backend/internal/service/chapter/usecase.go:16`  
   Severity: 高  
   Recommendation: 引入 service 层端口接口，infra 实现端口  
   Refactor Priority: P0
   完成情况：chapter service 依赖已切为 `llmGateway / txExecutor / extractionRunner / kgSyncer / jsonExecutor` 等本地端口接口，不再直接以 concrete infra/service 类型编排主路径。
   验证方式：`go test ./internal/service/chapter`、全量 `go test ./...`。

5. **来源：架构审查 11**  
   Finding: `chapter use case` 依赖与职责膨胀（God UseCase）  
   Location: `backend/internal/service/chapter/usecase.go:30`  
   Severity: 高  
   Recommendation: 按 `query/mutation/generation/review` 拆用例  
   Refactor Priority: P0
   完成情况：chapter 场景已按 `query / write / stream / review / context / extraction / update_tx` 等文件分治，`usecase.go` 收敛为组合与少量共用逻辑。
   验证方式：`go test ./internal/service/chapter`、全量 `go test ./...`。

6. **来源：性能优化审查 9**  
   Finding: 章节列表 SQL 返回 `content/summary` 大字段，列表 overfetch  
   Location: `backend/internal/infra/storage/postgres/chapter_repository.go:158`  
   Severity: 高  
   Recommendation: `list/detail DTO` 分离  
   Refactor Priority: P0
   完成情况：Postgres 与 memory 章节仓储均已新增 `ListListItemsByProject()`，`chapter-summaries` 走真实 summary DTO，不再通过完整章节列表映射。
   验证方式：`go test ./internal/infra/storage/postgres ./internal/infra/http/handler`、全量 `go test ./...`。

7. **来源：性能优化审查 10**  
   Finding: 资产列表 SQL 返回 `content/content_schema`，负载过重  
   Location: `backend/internal/infra/storage/postgres/asset_repository.go:63`  
   Severity: 高  
   Recommendation: `summary DTO + 详情按需取`  
   Refactor Priority: P0
   完成情况：资产列表已改为 `AssetListItem` 摘要 DTO，仓储新增 `ListListItemsByProject` / `ListListItemsByProjectAndType` / `ListTitlesByProjectAndType`，前端改为在编辑/大纲解析时按需 `getAsset()` 拉详情。
   验证方式：`go test ./internal/infra/storage/postgres ./internal/infra/http/handler`、`npm run test`、`npm run build`。

8. **来源：性能优化审查 11**  
   Finding: 章节记忆新鲜度检查 `250ms` DB 轮询，最长 `30s`  
   Location: `backend/internal/service/chapter/usecase.go:57`  
   Severity: 高  
   Recommendation: 事件驱动或指数退避  
   Refactor Priority: P0
   完成情况：`ensureChapterMemoryFresh()` 已改为 `500ms -> 1s -> 2s -> 4s -> 5s cap` 的指数退避等待，保留 `30s` 总超时与原有错误语义。
   验证方式：`go test ./internal/service/chapter` 覆盖 `nextMemoryStalePollInterval`，并通过全量 `go test ./...`。

9. **来源：性能优化审查 12**  
   Finding: 资产生成/优化前全量加载项目资产正文  
   Location: `backend/internal/service/asset/generation_runner.go:46`  
   Severity: 高  
   Recommendation: SQL 层先裁剪 + metadata 优先  
   Refactor Priority: P0
   完成情况：asset generate/refine 上下文已切到 `ListListItemsByProject()` + `BuildAssetListItemContextPack()`，只在 refine 目标详情上读取完整正文。
   验证方式：`go test ./internal/service/asset` 中 metadata-first 回归测试，以及全量 `go test ./...`。

10. **来源：性能优化审查 13**  
   Finding: SSE 写出链路把全文再聚合一份，长文本峰值内存放大  
   Location: `backend/internal/infra/http/handler/sse.go:34`  
   Severity: 高  
   Recommendation: 降低 handler 侧全文缓存  
   Refactor Priority: P0
    完成情况：`writeSSEStream()` 已退回纯传输层，不再在 handler 侧累计全文；全文与 token usage 累计已下沉到 `aiworkflow.StartStream()` 的 service-side proxy stream 中。
    验证方式：`go test ./internal/service/aiworkflow ./internal/service/asset ./internal/service/chapter ./internal/infra/http/handler`、全量 `go test ./...`。

11. **来源：设计模式评估 5**  
   Finding: `LoadBootstrap` 组合根膨胀  
   Location: `backend/internal/app/bootstrap.go:61`  
   Severity: 高  
   Pattern Concern: `God Factory`  
   Recommendation: 模块化初始化  
   Refactor Priority: P0
    完成情况：`bootstrap.go` 已引入 `appUseCases` 组合结构和 staged init helper，组合根不再直接内嵌所有构造细节。
    验证方式：`go test ./internal/app`、全量 `go test ./...`。

12. **来源：设计模式评估 7**  
   Finding: `ChapterHandler` 职责过载  
   Location: `backend/internal/infra/http/handler/chapter.go:21`  
   Severity: 高  
   Pattern Concern: `God Handler`  
   Recommendation: `CRUD/Generation/Review` 拆分  
   Refactor Priority: P0
    完成情况：chapter handler 已拆为 query / write / stream / review 子处理器，并保留原路由入口与行为不变。
    验证方式：`go test ./internal/infra/http/handler`、全量 `go test ./...`。

13. **来源：设计模式评估 9**  
   Finding: `chapter use case` 依赖 `18+` 且多协调器内聚在一个 struct  
   Location: `backend/internal/service/chapter/usecase.go:30`  
   Severity: 高  
   Pattern Concern: `God UseCase`  
   Recommendation: 子场景分治 + 薄 Facade  
   Refactor Priority: P0
    完成情况：chapter 用例当前通过场景化文件与局部端口接口组合，主入口依赖面已收窄，不再直接握住所有具体实现细节。
    验证方式：`go test ./internal/service/chapter`、全量 `go test ./...`。

14. **来源：设计模式评估 10**  
   Finding: `Brainstorm` 在 handler 层做并发编排  
   Location: `backend/internal/infra/http/handler/project.go:273`  
   Severity: 高  
   Pattern Concern: `Transport Orchestration Leakage`  
   Recommendation: 下沉 stream adapter  
   Refactor Priority: P0
    完成情况：brainstorm 的 goroutine/channel/SSE 写出已收敛到 `writeBrainstormStream()`，handler 本身只负责请求映射与 helper 调用。
    验证方式：`go test ./internal/infra/http/handler`、全量 `go test ./...`。

15. **来源：代码规范检查 12**  
   Finding: `LoadBootstrap` 超长且职责过载  
   Location: `backend/internal/app/bootstrap.go:61`  
   Severity: 高  
   Rule Violated: `SRP/可读性`  
   Recommendation: 五段式拆分  
   Refactor Priority: P0
    完成情况：`LoadBootstrap()` 已改为配置/迁移后串接 `initStorage()`、`initLLM()`、`initUseCases()`、`initHTTP()`，可读性与错误清理路径均已收束。
    验证方式：`go test ./internal/app`、全量 `go test ./...`。

16. **来源：代码规范检查 14**  
   Finding: `ExtractAfterGeneration` 为超长 orchestration  
   Location: `backend/internal/service/extraction/service.go:91`  
   Severity: 高  
   Rule Violated: 函数复杂度控制  
   Recommendation: 分 `prepare/build/admit/promote`  
   Refactor Priority: P0
    完成情况：提取链路当前按 `loadKnownCharacters / buildMemoryDerivation / admitMemoryDerivation / stageDerivationJob / promoteMemoryDerivation / markDerivationJobFailed` 分段执行，最新还将已知角色加载切到了 title-only 轻量查询。
    验证方式：`go test ./internal/service/extraction`、全量 `go test ./...`。

17. **来源：代码规范检查 15**  
   Finding: `Brainstorm handler` 并发/SSE 编排越界  
   Location: `backend/internal/infra/http/handler/project.go:273`  
   Severity: 高  
   Rule Violated: handler 薄层原则  
   Recommendation: 抽 stream helper  
   Refactor Priority: P0
    完成情况：brainstorm 流式事件写出已抽到独立 helper 文件，project ideation handler 只保留 request -> use case -> stream helper 的薄层装配。
    验证方式：`go test ./internal/infra/http/handler`、全量 `go test ./...`。

#### 中风险 / P1

1. **来源：代码冗余检测 12**  
   Finding: 多个 service 重复 `TrimSpace + ValidateUUID + ensureProjectExists + TranslateStorageError` 模板  
   Location: `backend/internal/service/characterstate/usecase.go:33`  
   Severity: 中  
   Recommendation: 抽共用验证/查验 helper  
   Refactor Priority: P1
   完成情况：已新增 `internal/service/project_scope.go`，并将 `characterstate / timeline / causallink` 中共用的 project/chapter scope 校验路径收敛到公共 helper。
   验证方式：`go test ./internal/service ./internal/service/timeline ./internal/service/causallink`、全量 `go test ./...`。

2. **来源：性能优化审查 14**  
   Finding: 提取服务只为角色名却加载全部角色正文  
   Location: `backend/internal/service/extraction/service.go:157`  
   Severity: 中  
   Recommendation: 新增轻量 `title/id` 查询  
   Refactor Priority: P1
   完成情况：`loadKnownCharacters()` 已改为调用 `ListTitlesByProjectAndType()`，不再为角色名聚合去读取完整角色正文。
   验证方式：`go test ./internal/service/extraction` 中 `TestLoadKnownCharactersUsesTitleOnlyQuery`，以及全量 `go test ./...`。

## Phase 4｜后端装配层、路由与基础设施治理

这一阶段处理后端装配层和基础设施侧的中低风险治理项。它们不会像 Phase 3 那样直接阻塞主链路，但如果长期堆积，会持续抬高理解和改动成本。放在核心用例拆分之后推进，可以避免在主骨架尚未稳定时过早调整路由入口、命名和轻量包装层。

### 修复目标

- 收敛 routes 注册、runtime service bag、模板化不完整和基础接口膨胀等问题。
- 清理死代码、空包装层和低收益维护点。
- 把命名一致性与基础 GoDoc 补齐到可持续状态。

### 预期产出

- 分域 registrar 化的路由组织方式。
- 更窄的 KG 仓储接口与更轻的装配暴露面。
- 基础注释、命名和死代码清理的收口清单。

### 验收标准

- 路由注册入口不再成为依赖堆叠中心。
- 基础设施层只暴露必要能力，不再保留长期空转或重复维护的结构。
- 命名与 GoDoc 至少在当前暴露边界上达到一致和可读。

### 完成产出

- `backend/api/http` 已按 health、stats、project/asset/chapter、memory、knowledge graph、LLM provider 拆为分域 registrar；`routes.go` 仅保留依赖收敛与顶层调度。
- `Bootstrap` 已收敛为最小运行时 surface：内部依赖改为未导出字段，对外仅保留 `Server()`、`WriteTimeoutSeconds()` 和 `Close()`；`cmd/server` 不再直接读取 runtime service bag。
- chapter handler 已补齐通用模板 helper：`bindChapterRequest`、`runChapterSingleCall`、`wrapChapterStreamCompletion` 统一了承载 `BindJSON + 单次 AI 超时包装 + 完成态响应整形` 的重复路径。
- KG 仓储接口已缩到真实使用面：domain interface 与 memory/postgres 双实现只保留 `CreateNode`、`ListNodes`、`CreateEdge`、`ListEdges`、`DeleteAllByProject`。
- `service/stats` 已新增 `ProjectStatsByID()`，项目 handler 不再自行从全量 stats 结果重复拼索引；prompt builder 构造命名、死代码和 `DebugConfig` GoDoc 也已同步收口。

### 验收结果

- 路由装配入口已不再承担所有路由细节：optional routes 的条件注册继续保留，但 domain-level registrar 已拆出，`RegisterRoutes` 仅负责编排。
- 基础设施与装配暴露面已收窄：`Bootstrap` 不再把 config / repo / LLM runtime 作为公共字段暴露，KG 仓储接口也不再保留未被 use case 使用的 CRUD 空能力。
- 命名与基础注释已回到一致状态：`NewPromptEnvelopeBuilder` 与 `PromptEnvelopeBuilder` 对齐，`DebugConfig` 补齐 GoDoc，chapter handler 的 sync/stream 模板也不再散落重复闭包。

### 完成状态

- 状态：`已完成（2026-04-01）`
- 后端验证：`go test ./internal/app ./internal/infra/http/handler ./internal/service/knowledgegraph ./internal/service/stats`、`go test ./...`、`go vet ./...`、`go build ./...`
- 验证结果：定向测试通过；后端全量测试通过；`go vet` 通过；后端全量构建通过。

### 纳入问题清单

#### 中风险 / P1

1. **来源：架构审查 7**  
   Finding: 路由注册和依赖在单一中心文件堆叠  
   Location: `backend/api/http/routes.go:23`  
   Severity: 中  
   Recommendation: 按域拆 registrar + 子依赖  
   Refactor Priority: P1
   完成情况：`backend/api/http/routes.go` 已收敛为顶层入口，新增 `route_handlers.go` 与分域 `routes_*.go` registrar 文件，health、stats、project/asset/chapter、memory、KG、LLM provider 路由注册已按域拆开。
   验证方式：`go test ./internal/infra/http/handler`、`go test ./...`。

2. **来源：代码冗余检测 9**  
   Finding: KG 仓储接口含大量无调用 CRUD，双实现重复维护  
   Location: `backend/internal/domain/knowledgegraph/repository.go:21`  
   Severity: 中  
   Recommendation: 缩窄接口到真实用例  
   Refactor Priority: P1
   完成情况：`knowledgegraph.Repository` 已缩窄为 `CreateNode / ListNodes / CreateEdge / ListEdges / DeleteAllByProject`；memory 与 postgres 仓储中无调用的 `UpdateNode / DeleteNode / GetNode / FindNodeByTypeAndName / DeleteEdge / FindEdge` 已删除。
   验证方式：`go test ./internal/service/knowledgegraph`、`go test ./...`。

3. **来源：设计模式评估 6**  
   Finding: `Bootstrap` 暴露 runtime service bag  
   Location: `backend/internal/app/bootstrap.go:49`  
   Severity: 中  
   Pattern Concern: `Service Bag`  
   Recommendation: 只暴露最小启动/关闭接口  
   Refactor Priority: P2
   完成情况：`Bootstrap` 的 config、server、LLM、repository、chapter use case 已改为未导出字段，对外新增 `Server()` 与 `WriteTimeoutSeconds()`，`cmd/server/main.go` 只消费最小运行时接口。
   验证方式：`go test ./internal/app`、`go test ./...`、`go build ./...`。

4. **来源：设计模式评估 8**  
   Finding: `sync/stream` 成对重复，模板化不完整  
   Location: `backend/internal/infra/http/handler/chapter.go:69`  
   Severity: 中  
   Pattern Concern: `Template Method 不完整`  
   Recommendation: 提炼统一执行模板  
   Refactor Priority: P1
   完成情况：chapter handler 已新增 `bindChapterRequest`、`runChapterSingleCall`、`wrapChapterStreamCompletion`，`chapter_write.go`、`chapter_review.go` 与大部分 `chapter_stream.go` 的重复模板已统一复用。
   验证方式：`go test ./internal/infra/http/handler`、`go test ./...`。

5. **来源：代码规范检查 11**  
   Finding: `RegisterRoutes` 过长  
   Location: `backend/api/http/routes.go:41`  
   Severity: 中  
   Rule Violated: 函数职责聚焦  
   Recommendation: 分域注册函数  
   Refactor Priority: P1
   完成情况：`RegisterRoutes()` 已改为“构建共享 handlers + 分域 registrar 调度”的薄入口，不再内联所有 route 和 optional dependency 分支。
   验证方式：`go test ./internal/infra/http/handler`、`go test ./...`。

6. **来源：代码规范检查 13**  
   Finding: 构造命名不一致 `NewPromptAssembler vs PromptEnvelopeBuilder`  
   Location: `backend/internal/service/prompt_assembler.go:20`  
   Severity: 中  
   Rule Violated: 命名一致性  
   Recommendation: 统一构造函数或类型名  
   Refactor Priority: P2
   完成情况：prompt builder 构造函数已统一更名为 `NewPromptEnvelopeBuilder`，并同步更新 `bootstrap`、`asset / chapter / project / extraction` 及相关测试调用点。
   验证方式：`go test ./internal/service ./internal/app`、`go test ./...`。

#### 低风险 / P2-P3

1. **来源：代码冗余检测 10**  
   Finding: `newProjectResponses` 死代码  
   Location: `backend/internal/infra/http/handler/project.go:152`  
   Severity: 低  
   Recommendation: 删除  
   Refactor Priority: P3
   完成情况：`internal/infra/http/handler/project.go` 中未被调用的 `newProjectResponses()` 已删除，项目 handler 仅保留当前真实使用的 response 构造路径。
   验证方式：`go test ./internal/infra/http/handler`、`go test ./...`。

2. **来源：代码冗余检测 11**  
   Finding: `stats usecase` 为空转包装层  
   Location: `backend/internal/service/stats/usecase.go:9`  
   Severity: 低  
   Recommendation: 合并层级或增加真实业务职责  
   Refactor Priority: P3
   完成情况：`service/stats` 已新增 `ProjectStatsByID()` 聚合 helper，项目 handler 改为直接消费项目级统计索引，`stats usecase` 不再只是单纯透传 `GetStats()`。
   验证方式：`go test ./internal/service/stats ./internal/infra/http/handler`、`go test ./...`。

3. **来源：代码规范检查 10**  
   Finding: `DebugConfig` 缺 GoDoc  
   Location: `backend/pkg/config/config.go:142`  
   Severity: 低  
   Rule Violated: `AGENTS 6.5`  
   Recommendation: 补注释  
   Refactor Priority: P3
   完成情况：`pkg/config/config.go` 中 `DebugConfig` 已补齐 GoDoc，说明其仅承载开发期诊断开关。
   验证方式：`go test ./internal/app ./pkg/config`、`go test ./...`。

## Phase 5｜跨模块契约、测试基线与最终收口

这一阶段放在最后，不是因为问题不重要，而是因为它们天然依赖前四个阶段的结构稳定性。共享契约、zero-warning 质量门、E2E 基线和共享 DTO 文档属于跨模块收口项，过早推进会在前后端结构持续调整时反复返工。等主路径与核心用例收束后，再统一建立质量门和契约边界，成本最低、结果也最稳定。

### 修复目标

- 收回前端对 generated contracts 的重复真相源。
- 在主路径结构稳定后建立真正可执行的 zero-warning 与 E2E 基线。
- 把跨模块共享 DTO 的文档与契约边界补全到可作为长期约束的程度。

### 预期产出

- `prompt-profile` 回归 generated contracts 单一真相源。
- `eslint` zero-warning 质量门明确，E2E 占位脚本要么补齐要么移除。
- 后端共享 DTO 的 GoDoc 和最终跨模块收口说明。

### 验收标准

- 前端不再围绕稳定共享协议保留二次构造的本地真相源。
- 质量门不是占位声明，而是能直接决定是否允许后续整改收口的真实门槛。
- 共享协议边界、契约生成链路和最终验收口径在文档中清晰一致。

### 纳入问题清单

#### 高风险 / P0

1. **来源：代码规范检查 1**  
   Finding: ESLint 规则允许 `warn`，与 `zero-warning` 冲突  
   Location: `frontend/eslint.config.js:22`  
   Severity: 高  
   Rule Violated: `AGENTS 5.1`  
   Recommendation: 改为 `error` 或消除触发点  
   Refactor Priority: P0

2. **来源：代码规范检查 9**  
   Finding: 共享 DTO 导出类型缺 GoDoc  
   Location: `backend/internal/contracts/httpdto/shared.go:9`  
   Severity: 高  
   Rule Violated: `AGENTS 6.5`  
   Recommendation: 给导出 DTO 分组补 GoDoc  
   Refactor Priority: P0

#### 中风险 / P1

1. **来源：代码冗余检测 6**  
   Finding: `prompt-profile.ts` 对 generated 契约二次构造，形成重复真相源  
   Location: `frontend/src/shared/api/prompt-profile.ts:29`  
   Severity: 中  
   Recommendation: 直接消费 generated schema/枚举  
   Refactor Priority: P1

2. **来源：代码冗余检测 8**  
   Finding: `@playwright/test + test:e2e` 仅预留，未落地资产  
   Location: `frontend/package.json:13`  
   Severity: 中  
   Recommendation: 补齐配置和用例，或删除脚本依赖  
   Refactor Priority: P2

## 收口说明

### 执行顺序

1. 先完成 `Phase 1`，避免前端主路径结构继续放大 shared 层与后端接口压力。
2. 再完成 `Phase 2`，把前端共享层和局部规范债务在结构稳定后一次性收口。
3. 进入 `Phase 3`，处理后端核心用例、热点查询和流式链路，给分页、懒加载和契约收敛提供稳定后端基础。
4. 接着执行 `Phase 4`，清理装配层、命名、死代码和低收益维护点。
5. 最后执行 `Phase 5`，统一关闭共享契约、质量门、E2E 基线与文档收口项。

### 阶段依赖

- `Phase 2` 依赖 `Phase 1` 给出稳定的页面与场景边界，否则 shared 层抽象容易重复返工。
- `Phase 3` 为 `Phase 5` 提供分页、查询和 handler/service 边界的稳定基础。
- `Phase 4` 需要在 `Phase 3` 核心骨架收束后推进，避免装配层治理反复调整。
- `Phase 5` 以“结构已定型”为前提，统一建立 shared contracts、zero-warning 与 E2E 基线。

### 最终验收口径

- 五个阶段的 62 条 finding 均有明确归属，且仅出现一次。
- 前后端整改继续遵守父仓库与子模块边界：业务代码在 `frontend/` 与 `backend/` 内提交，父仓库负责文档与子模块指针收口。
- 共享契约仍以后端稳定 DTO 为真相源，通过 `go run ./cmd/contractgen` 生成至前端 `generated contracts`。
- 最终收口不以“文档归档”为准，而以前端主路径稳定、后端核心分层回归、共享契约一致、质量门可执行为准。
