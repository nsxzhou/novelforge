# InkMuse AI 小说生成链路安全与稳定性重构方案

## Summary

现有实现已经具备完整业务链路，但核心问题不是"功能缺失"，而是"边界失真"。最重要的证据点是：

- 完整 prompt 会被序列化进入 generation record，并在部分前端界面展开显示，见  
  `backend/internal/service/generation_helpers.go:132` 和  
  `frontend/src/shared/ui/prompt-trace-card.tsx:57`
- POV 过滤目前只是提示语，不是真裁剪，见 `backend/internal/service/chapter/context.go:257`
- extraction 会把模型产出的摘要、角色状态、时间线、scratchpad 直接提升为后续上下文，见  
  `backend/internal/service/extraction/service.go:257`
- JSON repair retry 会把原始 prompt 再送回模型，见 `backend/internal/service/json_execution.go:259`

最优方案不是分别采纳两份计划中的"最大集合"，而是明确 6 个架构决策：

1. 用统一 PromptEnvelope 重建 prompt 信任边界
2. 用 PromptTraceManifest + PromptAuditSnapshot 重建 trace 与调试数据边界
3. 用"按操作类型路由 provider"的方式重建模型选择与上下文预算
4. 用 source_version + derivation_version + staged admission 重建记忆层、评审层与图谱同步
5. 用"真实 POV 裁剪 + degraded_pov_mode"替代当前伪过滤
6. 用"structured -> local repair -> single remote repair"替代当前高泄露 JSON 回退

交付按 1 后端 + 1 前端设计，采用 5 周主线 + 1 周缓冲。4 周不足以完成数据迁移、契约扩展、回归和性能收口。

## Key Changes

### 8. 质量链路与确认门禁

#### 自动触发

每次章节生成、续写、改写后自动触发：

- 最新 derivation
- chapter_review

#### Confirm 门禁固定

- `memory_stale` 禁止确认
- `derivation_failed` 禁止确认
- `chapter_review` 解析失败禁止确认
- `degraded_pov_mode` 强警示但不阻断
- review 低分只警示，不阻断

#### review 结果新增

- `data_risks`
- `continuity_risks`
- `provider_failover_used`
- `degraded_flags`

### 9. 前端流式与可观测性收口

#### 流式控制

- 所有章节流式面板统一到 `useStreamTask` 等价的防竞态控制语义
- sse-client 兼容 `event:` / `data:` 是否带空格、CRLF、chunk split

#### 前端统一展示

- `memory_stale`
- `derivation_running`
- `derivation_failed`
- `degraded_pov_mode`
- `provider_failover_used`

#### 指标新增

- prompt 组装耗时
- 首 token 时间
- review 耗时
- derivation 总耗时
- section 长度
- 裁剪率
- repair 率
- stale gate 命中率

## Public API / Data Changes

### prompt_trace

- 字段名保持不变
- 内容语义改为脱敏 manifest

### generation_record

- 保持现有响应形状
- `input_snapshot_ref` 改为 manifest
- `output_ref` 改为 opaque reference
- 新增 provider / routing / repair / derivation 相关字段

### llm_provider

- 新增 `operation_capabilities`

### chapter

- 新增 `pov_character_name`

### outline

- chapter seed 新增 `pov_character_name`

### derivation_status

新增：

- `memory_stale`
- `degraded_pov_mode`
- `source_version`
- `latest_derivation_version`
- `last_attempt_at`
- `last_success_at`
- `retryable`

### 新增表

- `chapter_derivation_jobs`
- `debug_prompt_captures`

## Test Plan

### Prompt 注入对抗测试

在 instruction、章节正文、资产正文、上一章尾段、scratchpad、brainstorm 输出中注入"忽略以上规则""输出非 JSON""改写 schema"等 payload，所有生成、评审、提取路径都必须继续满足既定 schema 与 policy。

### 泄露回归

普通前端页面、普通 API 响应、普通 generation record、普通日志中不得再出现完整 `rendered_system`、`rendered_user`、完整正文型 `output_ref`。

### Provider / 路由测试

provider 超时、4xx、5xx、structured unsupported、failover、非法 base_url、loopback/local provider 例外路径。

### Derivation 一致性测试

- 章节更新后 derivation 未完成时，再次生成/评审必须等待或显式失败
- scheduler 去抖、pending rerun、watchdog、重启恢复都补测试

### POV 正确性测试

- `first_person` / `third_limited` 且设置 `pov_character_name` 时，不得注入 POV 未知信息
- 缺少 `pov_character_name` 时必须标记 `degraded_pov_mode`

### JSON 回退测试

structured 成功、本地 repair 成功、远程 repair 成功、三者都失败四类路径全部覆盖。

### 前端流式测试

review、polish、rewrite-preview、suggest 的成功、取消、晚到回调、重试、错误提示补齐。

### 性能基线

prompt 装配时间、首 token 时间、review 耗时、derivation 耗时、context 长度、裁剪率建立基线。

## Timeline and Resources

### 第 1 周

**后端**

- generation record 扩展
- trace manifest
- 日志脱敏

**前端**

- PromptTraceCard 改 manifest 展示
- 资产/候选页适配

### 第 2 周

**后端**

- PromptEnvelope
- 严格模板渲染
- capability 数据白名单
- operation routing 落地

**前端**

- trace manifest
- provider/model
- degraded 信息展示

### 第 3 周

**后端**

- source_version
- chapter_derivation_jobs
- staged admission
- memory_stale gate
- JSON repair 重构

**前端**

- derivation stale/running/failed 交互

### 第 4 周

**后端**

- pov_character_name 契约
- 真实 POV 裁剪
- degraded_pov_mode
- review 数据风险字段

**前端**

- POV 配置入口
- review/confirm 联动

### 第 5 周

**后端**

- KG 查询优化
- scheduler watchdog
- 指标补全
- 回归修补

**前端**

- 流式面板统一控制器语义
- 状态收口

### 第 6 周

缓冲周，只做迁移修补、性能偏差修复和最终验收。

### 资源固定

- 后端 1 人全程负责 chapter / extraction / llm / promptbank / contracts / migration
- 前端 1 人全程负责 trace、stale gating、POV 配置、review/confirm 联动、流式交互回归

## Assumptions and Defaults

- 本项目定位为本地单用户应用，不考虑 authn/authz
- 路由路径保持不变，优先做字段扩展和语义迁移
- promptbank、SSE、双 provider 模式保留
- debug capture 默认关闭，仅本地显式开启
- 历史记录不做完整 prompt 内容迁移回灌；迁移后新写入记录使用新语义，旧记录按兼容方式读取
