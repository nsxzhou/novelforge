# InkMuse

InkMuse 是一个开源的单用户 AI 小说创作工具，在本地运行，提供从项目创建到章节生成的完整创作流程。

核心流程：**组合式或手动创建项目 → 设定/大纲管理 → AI 章节生成 → 编辑/改写 → 确认定稿**。

## 仓库结构

本仓库为父仓库，通过 Git submodule 管理前后端子仓库。

```text
InkMuse/
├── backend/                 # 后端子模块 — Go + Hertz + PostgreSQL
├── frontend/                # 前端子模块 — React + TypeScript + Tailwind CSS
├── 运行文档.md               # 运行文档
└── .gitmodules
```

### 前端

React 18 + Vite + TanStack Query + Tiptap 富文本编辑器。采用 scheme-05-minimal 极简设计（冷灰色调、纯 Inter 字体、无阴影渐变）。

主要功能：
- 项目仪表盘（统计卡片 + 看板视图）
- 组合式或手动创建项目
- 设定工坊（资产 CRUD + AI 生成 + 生成后预览编辑 + 结构化表单）
- 章节编辑（Tiptap 编辑器 + AI 续写/改写/Ghost Text + 未保存防误确认）
- 章节质量评审（AI 多维度评分面板）
- 记忆层（角色状态 + 时间线事件 + 关系图谱交互编辑）
- 知识图谱（角色-地点-事件-物品关系网络力导向图可视化 + 同步构建）
- 消痕润色（AI 文本风格优化 + SSE 流式 + 原文对比）
- 视角过滤（POV 角色选择 + 上下文自动过滤）
- 成本看板（token 消耗/生成次数/耗时统计 + CSS 柱状图）
- LLM Provider 配置
- 项目导出（Markdown / 纯文本 / EPUB）

### 后端

Go + Hertz HTTP 框架 + PostgreSQL。DDD 分层架构（领域层 / 服务层 / 存储层）。

主要功能：
- 项目、资产、章节 CRUD
- AI 生成/续写/改写（同步 JSON + SSE 流式）
- 章节质量评审（多维度 AI 评分）
- 消痕润色（AI 文本风格优化，SSE 流式）
- 视角信息过滤（POV 角色上下文自动过滤）
- 角色状态、时间线事件与章节后提取
- 因果链追踪（事件间因果关系映射）
- 知识图谱（角色-地点-事件-物品关系网络 + 同步构建）
- OpenAI 兼容 LLM 客户端（支持多 Provider）
- 编译内嵌 Prompt 模板管理
- 指标采集 + 成本看板（token 消耗/生成次数/耗时聚合统计）
- 项目导出（Markdown / 纯文本 / EPUB）

## 前后端交互方式

- 前端默认通过 `VITE_API_BASE_URL` 连接后端，开发环境默认值是 `http://127.0.0.1:8080/api/v1`。
- 普通 JSON 接口统一走 `frontend/src/shared/api/http-client.ts` 的 `request()`；导出下载走同一客户端里的 `requestRaw()`。
- 流式生成接口统一走 `frontend/src/shared/api/sse-client.ts`，使用 SSE 消费章节生成、续写、改写、灵感 brainstorm、资产生成等结果。
- 非 `GET` JSON 请求默认携带 `X-User-ID` 作为本地用户标识；这不是鉴权系统，只用于本地单用户场景下的操作归属。
- 当前重要契约约定：
  `character-states.relationships` 已是结构化数组 DTO。
  `GET /llm/providers` 返回 `api_key_masked`，写接口仍提交 `api_key`。
  前端共享资产 schema、核心 DTO、brainstorm schema 与默认关系类型配置来自后端生成的 `frontend/src/shared/api/generated/contracts.ts`。
- 路由与契约清单见 [backend/README.md](backend/README.md)、[frontend/README.md](frontend/README.md)。

## 文档分层

- 当前实现与运行方式：根 `README.md`、[运行文档](运行文档.md)、[backend/README.md](backend/README.md)、[frontend/README.md](frontend/README.md)

## 克隆

```bash
git clone --recurse-submodules https://github.com/nsxzhou/InkMuse.git
cd InkMuse
```

已克隆但缺少子模块时：

```bash
git submodule update --init --recursive
```

## 开发流程

子模块独立提交，父仓库跟踪版本指针：

```bash
# 在子仓库内开发并提交
cd frontend  # 或 backend
git add . && git commit -m "..." && git push

# 回到父仓库更新子模块指针
cd ..
git add frontend && git commit -m "chore: bump frontend submodule" && git push
```

拉取最新代码后同步子模块：

```bash
git pull && git submodule update --init --recursive
```

## 远程仓库

- 父仓库：<https://github.com/nsxzhou/InkMuse>
- 后端：<https://github.com/nsxzhou/InkMuse-backend>
- 前端：<https://github.com/nsxzhou/InkMuse-frontend>
