# InkMuse

InkMuse 是一个开源的单用户 AI 小说创作工具，在本地运行，提供从灵感构思到章节生成的完整创作流程。

核心流程：**灵感对话 → 项目创建 → 设定/大纲管理 → AI 章节生成 → 编辑/改写 → 确认定稿**。

## 仓库结构

本仓库为父仓库，通过 Git submodule 管理前后端子仓库。

```text
InkMuse/
├── backend/                 # 后端子模块 — Go + Hertz + PostgreSQL
├── frontend/                # 前端子模块 — React + TypeScript + Tailwind CSS
├── ui-designs/              # 产品原型图（scheme-05-minimal 风格，HTML）
├── docs/notes/              # 规划草稿、审查记录等参考资料（非当前实现真相来源）
├── 运行文档.md               # 运行文档
└── .gitmodules
```

### 前端

React 18 + Vite + TanStack Query + Tiptap 富文本编辑器。采用 scheme-05-minimal 极简设计（冷灰色调、纯 Inter 字体、无阴影渐变）。

主要功能：
- 项目仪表盘（统计卡片 + 看板视图）
- 灵感对话创建项目
- 设定工坊（资产 CRUD + AI 生成 + 生成后预览编辑 + 结构化表单）
- 章节编辑（Tiptap 编辑器 + AI 续写/改写/Ghost Text + 未保存防误确认）
- 记忆层（角色状态 + 时间线事件）
- Prompt 模板管理、LLM Provider 配置
- 项目导出（Markdown / 纯文本）

### 后端

Go + Hertz HTTP 框架 + PostgreSQL。DDD 分层架构（领域层 / 服务层 / 存储层）。

主要功能：
- 项目、资产、章节 CRUD
- AI 生成/续写/改写（同步 JSON + SSE 流式）
- 角色状态、时间线事件与章节后提取
- OpenAI 兼容 LLM 客户端（支持多 Provider）
- Prompt 模板管理（编译内嵌 + 项目级覆盖 + 变量白名单校验）
- 指标采集（操作耗时、成功/失败事件）

## 文档分层

- 当前实现与运行方式：根 `README.md`、[运行文档](运行文档.md)、[backend/README.md](backend/README.md)、[frontend/README.md](frontend/README.md)
- 规划草稿与历史审查：`docs/notes/`，仅作为参考资料，不作为当前实现说明

## 克隆

```bash
git clone --recurse-submodules https://github.com/nsxzhou/inkmuse.git
cd inkmuse
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

- 父仓库：<https://github.com/nsxzhou/inkmuse>
- 后端：<https://github.com/nsxzhou/inkmuse-backend>
- 前端：<https://github.com/nsxzhou/inkmuse-frontend>
