# InkMuse AGENTS（LLM 规范）

本文件是 `InkMuse/` 根目录下的默认规则集，核心用途是指导大型语言模型（LLM）在本仓库内稳定地改代码：边界清晰、真相唯一、拒绝冗余。  
若未来在 `frontend/` 或 `backend/` 内出现更深层 `AGENTS.md`，以更深层文件为准。

## 0. 三条金律（必须遵守）

1) 契约与边界清晰：先确认模块职责、依赖方向与接口约定，再写代码；禁止跨层泄漏、禁止把编排逻辑塞进不该承担的层。
2) 单一真相源（SSOT）：共享数据结构、schema、枚举、查询 key、错误映射必须只有一个权威来源；生成代码只读。
3) 拒绝冗余：拒绝任何形式的重复（代码/逻辑/文档）；写之前先查仓库已有实现与既有约定，能复用就复用，能链接就链接。

## 1. 仓库边界（父仓库 / 子模块）

- 本仓库是父仓库；`frontend/` 与 `backend/` 通过 Git submodule 管理。
- 父仓库的职责：总览文档、运行说明、脚本、规范文件、子模块指针与跨仓约定。
- 业务改动应发生在子模块内：把 `frontend/` 或 `backend/` 当成独立仓库工作；父仓库仅提交 submodule 指针变更。
- 与运行方式、路由与交互方式有关的叙述信息以根 README 与子模块 README 为准：  
  [README.md](README.md) · [frontend/README.md](frontend/README.md) · [backend/README.md](backend/README.md)

## 2. Contracts & Boundaries（职责与依赖方向）

### 2.1 后端分层职责（Go / DDD）

| 层级 | 目录 | 负责 | 禁止 |
|---|---|---|---|
| HTTP 路由 | `backend/api/http/` | 路由注册与版本化 | 写业务逻辑 |
| HTTP handler | `backend/internal/infra/http/handler/` | 参数绑定、调用用例、写回响应、错误码映射 | 堆业务规则、直接操作存储实现 |
| 应用用例/编排 | `backend/internal/service/` | 用例编排、输入清洗、业务校验、错误语义、事务边界 | 泄漏存储细节到 domain；绕过统一错误映射 |
| 领域模型 | `backend/internal/domain/` | 实体/值对象/派生逻辑、仓储接口、不变式 | 依赖 infra；承载 HTTP 绑定细节 |
| 基础设施 | `backend/internal/infra/` | HTTP/LLM/存储/日志等实现 | 成为业务规则主承载层 |
| 稳定复用包 | `backend/pkg/` | 可跨内部模块复用的稳定能力 | 夹带具体业务编排 |
| 工具入口 | `backend/cmd/` | server、迁移、契约生成等工具入口 | 变成业务库 |

### 2.2 前端分层职责（React）

| 层级 | 目录 | 负责 | 禁止 |
|---|---|---|---|
| 页面入口 | `frontend/src/pages/` | 路由参数接入、主查询/主 mutation、视图装配 | 把复杂副作用或状态机散落在 JSX |
| 业务模块 | `frontend/src/features/<domain>/` | domain UI、局部状态、可复用面板、业务 hooks | 在页面与 feature 之间复制同一套逻辑 |
| 跨域共享 | `frontend/src/shared/` | API 客户端、共享类型边界、基础 UI、通用工具、环境配置 | 把“页面一次性逻辑”抽进 shared |
| 应用装配 | `frontend/src/app/` | 路由装配、QueryClient、全局样式与 token | 把业务细节堆进 app 层 |

通用落地规则（前后端一致）：
- 入口层只做接入与编排；细节下沉到可复用的 hook/service/helper/use case。
- 若某条规则/文案/query key/schema 会被多处使用，必须抽到集中位置，禁止复制粘贴扩散。

### 2.3 前后端接口约定（只写“契约”）

- JSON 错误响应统一输出：`error` + 可选 `error_code`。
- SSE 事件语义：
  - `content`：增量文本块（保持纯文本块语义）
  - `done` / `result` / `error`：终态事件必须与共享 schema 对齐
- 客户端调用入口（作为约束而非说明书）：
  - 普通 JSON 请求统一通过 `frontend/src/shared/api/http-client.ts`
  - SSE 流式统一通过 `frontend/src/shared/api/sse-client.ts`
- 交互与请求头等细节以根 README 为权威描述（避免文档重复）：[README.md](README.md)

## 3. Single Source of Truth（权威来源与强制流程）

### 3.1 SSOT 地图（改哪里、谁消费、怎么改）

| 领域/能力 | 真相源（SSOT） | 消费方 | 禁止 |
|---|---|---|---|
| 共享 HTTP DTO / 枚举 / 结构化 schema | `backend/internal/contracts/httpdto/` 等（后端先定义） | `contractgen` 生成 → 前端消费 | 前端手写同名同义 DTO；手改生成文件 |
| 生成契约输出 | `frontend/src/shared/api/generated/contracts.ts` | 前端所有稳定共享协议 | 任何手工编辑 |
| 前端共享类型边界（增强型） | `frontend/src/shared/api/types.ts` | UI 层 | 把增强类型变成共享协议真相源 |
| 前端查询 key | `frontend/src/shared/api/queries.ts` | queries / invalidation / 页面 | 页面内散写字符串数组 |
| 用户可见错误文案映射 | `frontend/src/shared/lib/error-message.ts` | UI | 各处重复写错误文案分支 |
| 设计 token（颜色等） | `frontend/src/app/styles.css` | 全站样式 | 发明第二套 token 命名体系 |

### 3.2 共享契约变更的强制流程（必须按顺序）

当你要变更共享字段/枚举/结构化 schema（会被前后端共同消费）时：
1) 先在后端定义/变更（共享协议真相源在后端）。
2) 通过 `backend/cmd/contractgen` 生成前端契约输出。
3) 前端只做“消费适配”，优先直接引用 generated contracts 中的类型或 schema。
4) 同步补齐回归测试：至少覆盖契约消费、错误映射、SSE 终态事件或迁移/存储映射等受影响路径。

## 4. Anti-Redundancy（拒绝冗余：给 LLM 的决策树）

### 4.1 写代码前：先查再写

新增任何“工具函数 / schema / 错误映射 / query key / SSE 封装 / helper”之前：
- 先在仓库内搜索是否已有同类实现或近似入口（同一能力只能有一个权威入口）。
- 已存在：复用或扩展既有实现，禁止新建平行版本。
- 不存在但会复用（复用点 ≥ 2 且边界清晰）：抽到集中位置（前端多半在 `shared/`，后端多半在 `pkg/` 或 `internal/service` 的公共模块），并让调用点收敛。
- 只是一处使用：允许局部实现，但禁止为“看起来更优雅”提前引入新范式。

### 4.2 拒绝逻辑冗余（同一规则只能实现一次）

- 输入清洗与校验：后端 service 层做入口清洗与业务校验，复用 `ValidateUUID` / `WrapInvalidInput` 等既有模式，禁止各处手写一套。
- 错误语义与映射：后端仓储错误统一翻译为 service 语义错误；前端用户可见文案统一走错误映射函数，禁止散落的 `if (err)` 文案拼装。
- 流式任务生命周期：前端 SSE 生命周期必须封装为共享客户端/hook，不允许散落在页面 JSX 中。
- LLM JSON 抽取/修复：后端集中在 `backend/internal/service/json_response.go` 与 `backend/internal/service/json_execution.go`，禁止在调用方复制解析/修复逻辑；背景与方案见 `backend/docs/llm-json-error-handling.md`。

### 4.3 拒绝文档冗余（规则写这里，细节写别处）

- AGENTS.md 只保留：边界、契约、SSOT、禁止事项、强制流程与检查清单。
- 运行说明、功能清单、路由清单、交互细节：用链接指向根 README / 子模块 README / `backend/docs`，禁止在 AGENTS 中复制一份造成漂移。

## 5. 代码风格与最小约束（只保留 LLM 必须记住的）

### 5.1 前端（TypeScript / React）

- 路径导入：优先 `@/` 指向 `frontend/src`，避免深层相对路径穿透。
- 命名与文件：kebab-case；优先命名导出；类型优先 `type`。
- 格式：单引号、无分号，遵循现有 formatter 结果。
- 契约：优先直接消费 `generated/contracts.ts`；禁止手写同名同义共享 DTO。

### 5.2 后端（Go）

- 分层：handler 只做绑定/调用/响应；业务编排在 `internal/service`；不变式在 `internal/domain`；实现细节在 `internal/infra`。
- 构造与注入：接口常用 `UseCase`；依赖结构体优先 `Dependencies`；构造函数优先 `NewXxx...`。
- 输入/错误/时间：入口字符串清洗（`strings.TrimSpace`）、UUID 校验复用 `ValidateUUID`、非法输入复用 `WrapInvalidInput`、存储错误复用 `TranslateStorageError`、时间用 `time.Now().UTC()`、主键用 `uuid.NewString()`。

## 6. 完成前检查（提交前自检清单）

- 是否明确写出了“改动点的边界”：该逻辑属于哪一层/模块，接口是什么。
- 是否遵守 SSOT：没有新增第二份 DTO/schema/query key/error mapping；没有手改生成文件。
- 是否消除了冗余：没有复制粘贴扩散；可复用规则已收敛到集中位置。
- 是否保持最小改动面：沿用既有目录分层、命名、错误处理方式与依赖方向。
- 是否需要同步测试或文档链接更新（而不是再写一份重复文档）。
