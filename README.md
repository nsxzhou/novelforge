# NovelForge

NovelForge 是一个开源、可扩展、可自部署的 AI 小说创作平台。

当前仓库是 **父仓库**，用于管理项目文档、整体结构和前后端子模块。V1 目标聚焦在“灵感 -> 项目创建 -> 设定/大纲 -> 单章草稿 -> 当前稿确认”的最小闭环。

## 相关文档

- 产品规划：`novelforge-product-plan.md`
- 开发优先级：`开发优先级.md`
- 运行文档：`运行文档.md`

## 仓库结构

```text
NovelForge/
├── backend/                     # 后端子模块：novelforge-backend
├── frontend/                    # 前端子模块：novelforge-frontend
├── novelforge-product-plan.md   # 产品规划
├── 开发优先级.md                # 开发优先级
└── .gitmodules                  # 子模块配置
```

### 子仓库说明

- `backend/`：独立 Git 仓库，当前已完成 Hertz + Eino 后端基础设施、项目/资产 API、Project / Asset 对话微调、章节生成 / 当前稿确认 / 续写 / 局部重写 HTTP API、GenerationRecord 持久化，以及 OpenAI 兼容 LLM 客户端与 Prompt 模板基础设施
- `frontend/`：独立 Git 仓库，采用侧边栏 + 工作区布局，已完成灵感对话创建项目、资产 CRUD 与 AI 生成、项目/资产对话微调确认写回、章节生成/续写/局部改写/当前稿确认

## 克隆项目

首次克隆请带上子模块：

```bash
git clone --recurse-submodules https://github.com/nsxzhou/novelforge.git
cd novelforge
```

如果已经普通克隆过父仓库，再执行：

```bash
git submodule update --init --recursive
```

## 更新项目

拉取父仓库最新内容后，同步子模块到父仓库记录的版本：

```bash
git pull
git submodule update --init --recursive
```

## 日常开发流程

### 开发后端

先在后端子仓库内提交：

```bash
cd backend
git pull
git add .
git commit -m "..."
git push
```

再回到父仓库，提交子模块指针更新：

```bash
cd ..
git add backend
git commit -m "chore: bump backend submodule"
git push
```

### 开发前端

先在前端子仓库内提交：

```bash
cd frontend
git pull
git add .
git commit -m "..."
git push
```

再回到父仓库，提交子模块指针更新：

```bash
cd ..
git add frontend
git commit -m "chore: bump frontend submodule"
git push
```

## 子模块注意事项

- `backend/` 和 `frontend/` 是 **submodule**，不是父仓库里的普通源码目录
- 修改前后端代码时，应先在各自子仓库提交并推送
- 父仓库提交的是子模块版本指针，而不是子仓库内部源码变更
- 如果拉取了父仓库的新提交，记得执行：

```bash
git submodule update --init --recursive
```

## 远程仓库

- 父仓库：<https://github.com/nsxzhou/novelforge>
- 后端仓库：<https://github.com/nsxzhou/novelforge-backend>
- 前端仓库：<https://github.com/nsxzhou/novelforge-frontend>
