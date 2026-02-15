# 剧本生成子系统 — 实现状态总览

> 最后更新：2026-02-15
> 工程仓库：`murder-mystery-generator`（通过 git submodule 引用本知识库）

## 项目概况

本项目是「线上剧本杀AI生成工具」的第一个子系统实现——**剧本生成子系统**（creation-service）。采用 pnpm monorepo 结构，包含 `packages/shared`（共享类型，已发布为 `@gdgeek/murder-mystery-shared`）、`packages/server`（后端服务）、`packages/web`（前端包，备用）。

### 技术栈

- Node.js + Express + TypeScript（pnpm monorepo）
- MySQL 8.0 + Redis
- Vitest + fast-check（属性测试）
- Docker Compose（开发 + 生产部署）
- GitHub Actions CI/CD（测试 + Docker 镜像推送到腾讯云）

### 测试覆盖

- 全部 447+ 单元测试通过
- 属性测试（PBT）覆盖核心正确性属性
- ESLint 代码检查集成到 CI

## 已实现功能（8 个 Spec）

### 1. 剧本生成核心（script-generation）✅ 完成

核心生成引擎，将用户配置参数转化为完整剧本杀内容。

**已实现组件：**
- `ConfigService` — 参数校验、轮次结构自动适配（2-6小时 → 2-5轮）、新本格特殊设定
- `SkillService` — JSON 文件加载 Skill 模板（honkaku/shin-honkaku/henkaku/common）
- `LLMAdapter` — 统一 LLM 接口、指数退避重试（3次）、Token 用量记录
- `GeneratorService` — LLM 提示词组装、JSON 解析、结构校验（玩家手册数量/线索一致性/分支可达性）
- `TagService` — 自动标签生成（5类）、自定义标签、Redis 缓存热门标签
- 反馈驱动优化 — 低分维度注入提示词、自动优化触发（≥5评价且任一维度<6分）
- 版本管理 — 优化创建新版本（v1.0→v1.1），原版本不可变

**数据模型：** script_configs、scripts、tags、script_tags 表


### 2. 分阶段创作工作流（staged-authoring）✅ 完成

灵感来源于 Kiro 的规格驱动开发，将一键生成拆分为三阶段结构化创作。

**工作流：**
- 企划阶段（Planning）→ 审阅/编辑/批准 → 大纲阶段（Design）→ 审阅/编辑/批准 → 生成阶段（Execution，按章节逐步生成）
- 同时保留一键生成模式（Vibe Mode）

**已实现组件：**
- `SessionStateMachine` — 状态机（draft → planning → plan_review → designing → design_review → executing → chapter_review → completed），含 failed 状态和重试
- `PromptBuilder` — 企划/大纲/章节提示词构建，注入前序阶段内容和作者备注
- `PhaseParser` — 解析 LLM 返回的 JSON 为 ScriptPlan/ScriptOutline/Chapter
- `AuthoringService` — 会话管理、阶段推进、编辑追踪、章节重新生成、最终组装
- 并行批量生成 — 多个玩家手册并行调用 LLM

**数据模型：** authoring_sessions 表（JSON 列存储阶段产出物）

**REST API：**
- `POST /api/authoring-sessions` — 创建会话
- `GET /api/authoring-sessions/:id` — 查询状态
- `POST /api/authoring-sessions/:id/advance` — 推进阶段
- `PUT /api/authoring-sessions/:id/phases/:phase/edit` — 编辑
- `POST /api/authoring-sessions/:id/phases/:phase/approve` — 批准
- `POST /api/authoring-sessions/:id/assemble` — 组装最终剧本

### 3. 多模型路由（multi-model-routing）✅ 完成

支持按任务类型分配不同 LLM 模型，跨提供商 Fallback。

**已实现组件：**
- `ConfigLoader` — 从 `config/llm-routing.json` 加载路由配置，支持环境变量覆盖 API Key
- `LLMRouter` — 实现 ILLMAdapter 接口，按 TaskType 路由到对应 Provider
- `LanguageDirectives` — 多语言提示词注入（en/zh）
- Fallback 链 — 主模型可重试错误耗尽后按优先级尝试备选模型
- 向后兼容 — 无配置文件时回退到环境变量单提供商模式

**TaskType 枚举：** planning、design、chapter_generation、one_shot_generation、optimization、default

### 4. 临时 AI 配置（ephemeral-ai-config）✅ 完成

服务器未预配置 AI 时，用户可临时输入 AI 信息，仅用于当前会话。

**已实现组件：**
- `AiStatusService` — 检测服务器 AI 配置状态（configured/unconfigured），含启动时连通性验证
- 会话级临时适配器 — `sessionAdapters: Map<string, ILLMAdapter>`，会话结束自动清理
- API Key 安全 — 不持久化 apiKey，数据库仅保留 provider/model 元信息
- 前端 AI 配置表单 — Provider 下拉（豆包/DeepSeek/智谱GLM/OpenAI/Anthropic/自定义）、自动填充默认值
- 连通性验证 — `POST /api/ai-status/verify`，创建会话前先验证

**REST API：**
- `GET /api/ai-status` — 检测 AI 配置状态
- `POST /api/ai-status/verify` — 验证临时配置连通性

### 5. 会话韧性与错误恢复（session-resilience）✅ 完成

确保用户在任何出错场景下都能优雅恢复，不丢失已有成果。

**已实现功能：**
- 每步 Token 报告 — `lastStepTokens` 记录每次 LLM 调用的 token 用量
- Session ID 显性化 — UI 显眼展示 + 一键复制 + URL hash 保存
- 会话恢复 — 输入 session ID 恢复之前的创作进度
- 会话中更换 AI 配置 — `PUT /api/authoring-sessions/:id/ai-config`（仅 failed/review 状态允许）
- 并行批量部分失败恢复 — 保留成功章节，仅重试失败部分
- 检查点保存 — 产出物先持久化再转换状态
- API Key 前置校验 — 路由层同步校验，无效 Key 直接返回 400

**REST API：**
- `PUT /api/authoring-sessions/:id/ai-config` — 更换 AI 配置
- `POST /api/authoring-sessions/:id/retry-failed-chapters` — 重试失败章节

### 6. OpenAPI 集成（openapi-integration）✅ 完成

Swagger UI + JSDoc 注解覆盖所有 API 端点。

**已实现：**
- 22+ 个 API 端点的 OpenAPI 3.0 注解
- Swagger UI 可通过 `/api-docs` 访问
- 22+ 个共享 Schema 定义（与 TypeScript 类型对齐）
- 中文描述、错误响应规范

### 7. Bootstrap 前端 UI（bootstrap-frontend-ui）✅ 完成

`packages/web` 包，Vite + TypeScript + Bootstrap 5 CDN。

**已实现：**
- Hash 路由器（#/、#/config、#/scripts）
- API 客户端封装
- 配置表单（含新本格特殊设定条件显示、比例滑块联动）
- 轮次结构预览
- 生成任务跟踪（轮询 + spinner）
- 剧本列表（分页卡片）

### 8. Token 用量追踪（token-usage-tracking）🔲 未开始

会话级累计 Token 用量，UI 实时展示。Spec 已创建，任务未执行。

> 注：`lastStepTokens`（每步 token）已在 session-resilience 中实现，本 spec 聚焦累计用量（CumulativeTokenUsage）。

## 额外实现（非 Spec 驱动）

### 内嵌测试 UI

`packages/server/src/routes/ui/` 下的暗色玻璃拟态风格 Web 面板，直接由 Express 服务。

**功能：**
- 分步创作完整流程（企划 → 大纲 → 章节 → 组装）
- AI 配置模态窗口（未配置时自动弹出，错误时自动弹出换 Key）
- 导航栏 AI 模型徽章（区分环境配置/用户输入）
- Session ID 显示 + 复制 + 恢复入口
- Token 用量展示（lastStepTokens + 累计）
- 失败恢复流程（重试/换 AI 配置/重试失败章节）
- 工作日志查看（原始记录 + 每日日记）
- 剧本导出（JSON，含 AI 用量摘要）
- Bug 报告入口（GitHub Issues）

### 7 种叙事风格系统

| 风格 | 代号 | 特色 |
|------|------|------|
| 悬疑 | Detective 正统侦探 | 严密逻辑推理，冷静克制 |
| 搞笑 | Drama 戏影侦探 | 谐音梗、无厘头、喜剧反转 |
| 探索 | Discover 寻迹侦探 | 多分支多结局，高可重玩性 |
| 浪漫 | Destiny 命运侦探 | 命运交织，浪漫情感 |
| 叙诡 | Dream 幻梦侦探 | 梦幻叙事，叙述性诡计 |
| 科幻 | Dimension 赛博侦探 | 高科技设定 |
| 恐怖 | Death 幽冥侦探 | 民俗/日式/哥特/克苏鲁恐怖 |

### DevOps

- Dockerfile（多阶段构建）
- docker-compose.prod.yml（生产部署，MySQL + Redis + Server）
- GitHub Actions CI（测试 + ESLint + Docker 镜像推送到腾讯云 hkccr.ccs.tencentyun.com）
- 自动数据库迁移（启动时执行未跑过的 SQL）
- 环境变量启动校验（非 ASCII、占位符检测）

### 工作日志系统

- Agent Hook 自动记录每次问答日志（按日期分文件）
- 手动触发生成每日工作日记（含工作评价和改进建议）
- UI 内查看工作日志

### 共享包发布

`@gdgeek/murder-mystery-shared` 发布到 GitHub Packages，包含所有共享类型定义和校验工具。


## 项目结构

```
murder-mystery-generator/
├── packages/
│   ├── shared/                         # @gdgeek/murder-mystery-shared（已发布 npm）
│   │   └── src/
│   │       ├── types/                  # 共享类型定义
│   │       │   ├── config.ts           # ScriptConfig, GameType, AgeGroup, RoundStructure
│   │       │   ├── script.ts           # Script, DMHandbook, PlayerHandbook, Material, BranchStructure, LLMRequest/Response, TokenUsage
│   │       │   ├── authoring.ts        # AuthoringSession, SessionState, ScriptPlan, ScriptOutline, Chapter, PhaseOutput, CumulativeTokenUsage
│   │       │   ├── ai-config.ts        # EphemeralAiConfig, AiStatusResult, AiVerifyResult, PROVIDER_DEFAULTS
│   │       │   ├── routing.ts          # RoutingConfig, ProviderConfig, TaskRoute, TaskType
│   │       │   ├── tag.ts              # Tag, TagCategory, ScriptTag
│   │       │   └── feedback.ts         # Feedback, AggregatedFeedback
│   │       └── validators/
│   │           └── ai-config-validator.ts  # validateEphemeralAiConfig
│   ├── server/                         # 后端服务
│   │   └── src/
│   │       ├── adapters/               # LLM 适配器层
│   │       │   ├── llm-adapter.ts      # LLMAdapter（单提供商，含重试）
│   │       │   ├── llm-router.ts       # LLMRouter（多模型路由 + Fallback）
│   │       │   ├── config-loader.ts    # 路由配置加载/校验
│   │       │   └── language-directives.ts  # 多语言提示词
│   │       ├── services/
│   │       │   ├── config.service.ts   # 配置校验 + 轮次结构
│   │       │   ├── skill.service.ts    # Skill 模板管理
│   │       │   ├── generator.service.ts # 剧本生成引擎
│   │       │   ├── tag.service.ts      # 标签系统
│   │       │   ├── ai-status.service.ts # AI 状态检测
│   │       │   └── authoring/          # 分阶段创作
│   │       │       ├── authoring.service.ts  # 核心服务
│   │       │       ├── state-machine.ts      # 状态机
│   │       │       ├── prompt-builder.ts     # 提示词构建
│   │       │       └── phase-parser.ts       # 阶段解析
│   │       ├── routes/                 # API 路由
│   │       │   ├── configs.ts          # POST/GET /api/configs
│   │       │   ├── scripts.ts          # 剧本 CRUD + 生成 + 导出
│   │       │   ├── tags.ts             # 标签 CRUD
│   │       │   ├── authoring.ts        # 创作会话全流程
│   │       │   ├── ai-status.ts        # AI 状态 + 验证
│   │       │   ├── work-log.ts         # 工作日志 API
│   │       │   └── ui/                 # 内嵌测试 UI
│   │       │       ├── body.html
│   │       │       ├── styles.css
│   │       │       └── app.js
│   │       ├── db/
│   │       │   ├── migrations/         # 5 个 SQL 迁移文件
│   │       │   ├── migrator.ts         # 自动迁移执行器
│   │       │   ├── mysql.ts
│   │       │   └── redis.ts
│   │       ├── skills/                 # Skill 模板 JSON
│   │       │   ├── common.json
│   │       │   ├── honkaku.json
│   │       │   ├── henkaku.json
│   │       │   └── shin-honkaku.json
│   │       ├── config/
│   │       │   └── env-validator.ts    # 启动时环境变量校验
│   │       ├── swagger.ts              # OpenAPI 配置 + Schema
│   │       ├── app.ts                  # Express 应用
│   │       └── server.ts               # 启动入口（迁移 → 启动）
│   └── web/                            # 前端包（Bootstrap 5 + Vite）
├── config/
│   └── llm-routing.example.json        # 路由配置示例
├── design-kb/                          # git submodule → 设计知识库
├── .kiro/
│   ├── specs/                          # 8 个功能 Spec
│   ├── steering/                       # 开发约定
│   ├── hooks/                          # Agent Hooks（日志记录 + 日记生成）
│   └── work-log/                       # 工作日志
├── .github/workflows/ci.yml           # CI/CD
├── Dockerfile                          # 多阶段构建
├── docker-compose.prod.yml             # 生产部署
└── eslint.config.js                    # ESLint 配置
```

## API 端点汇总（22 个）

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/health` | 健康检查 |
| POST | `/api/configs` | 创建剧本配置 |
| GET | `/api/configs/:id` | 获取配置详情 |
| POST | `/api/scripts/generate` | 生成剧本（异步） |
| GET | `/api/scripts/jobs/:jobId` | 查询生成任务状态 |
| GET | `/api/scripts` | 剧本列表 |
| GET | `/api/scripts/:id` | 剧本详情 |
| GET | `/api/scripts/:id/versions` | 版本历史 |
| POST | `/api/scripts/:id/optimize` | 手动触发优化 |
| GET | `/api/scripts/:id/export` | 导出剧本 JSON |
| GET | `/api/tags` | 标签列表 |
| GET | `/api/tags/popular` | 热门标签 |
| POST | `/api/scripts/:id/tags` | 添加自定义标签 |
| DELETE | `/api/scripts/:id/tags/:tagId` | 移除标签 |
| POST | `/api/authoring-sessions` | 创建创作会话 |
| GET | `/api/authoring-sessions/:id` | 查询会话状态 |
| POST | `/api/authoring-sessions/:id/advance` | 推进阶段 |
| PUT | `/api/authoring-sessions/:id/phases/:phase/edit` | 编辑阶段内容 |
| POST | `/api/authoring-sessions/:id/phases/:phase/approve` | 批准阶段 |
| PUT | `/api/authoring-sessions/:id/ai-config` | 更换 AI 配置 |
| POST | `/api/authoring-sessions/:id/retry-failed-chapters` | 重试失败章节 |
| POST | `/api/authoring-sessions/:id/assemble` | 组装最终剧本 |
| GET | `/api/ai-status` | AI 配置状态 |
| POST | `/api/ai-status/verify` | 验证临时 AI 配置 |

## 数据库迁移

| 文件 | 内容 |
|------|------|
| 001-init.sql | script_configs、scripts、tags、script_tags 表 |
| 002-authoring-sessions.sql | authoring_sessions 表 + 索引 |
| 002-add-style.sql | scripts 表添加 style 列 |
| 003-parallel-batch.sql | authoring_sessions 添加 parallel_batch 列 |
| 004-ai-config-meta.sql | authoring_sessions 添加 ai_config_meta 列 |
| 005-session-resilience.sql | authoring_sessions 添加 last_step_tokens 列 |

## 与微服务架构的对应关系

本项目当前以独立模式运行，对应微服务架构中的 `creation-service`（端口 3002）。

**已覆盖的 creation-service 功能：**
- ✅ 配置 CRUD
- ✅ 剧本生成 + 列表 + 详情 + 版本 + 优化
- ✅ 标签 CRUD + 搜索
- ✅ 分阶段创作工作流（超出原设计，是本项目的创新）

**未覆盖（属于其他子系统或未来扩展）：**
- 🔲 技能牌系统（Skill Cards / Designer Deck / Custom Deck）
- 🔲 访谈系统（Pre-Generation Interview）
- 🔲 知识库 RAG 集成
- 🔲 微服务模式路由前缀（`/api/creation/`）
- 🔲 内部 API（`/api/internal/`）
- 🔲 事件总线订阅

## 已知问题与技术债

1. 豆包 doubao-seed-2-0-pro 不支持 `response_format: json_object`，已改为默认不发送该参数
2. UI 前端 app.js 中 DOM 查询函数 `$`/`$$` 曾多次混用导致 bug，已修复并加 JSDoc 注释
3. `token-usage-tracking` spec 的累计 Token 用量功能尚未实现
4. 可选的属性测试（PBT）大部分未执行，核心单元测试已覆盖
5. 前端 `packages/web` 包功能较基础，主要使用内嵌测试 UI

## 下一步开发建议

按微服务架构规划，后续子项目可按以下优先级开发：

1. **knowledge-service** — 知识库子系统（向量存储 + RAG + 学习管道）
2. **auth-service** — 认证与账户
3. **gameplay-service** — 线上游玩（WebSocket + AI DM）
4. **feedback-service** — 反馈收集与权重计算
5. **ai-toolchain-service** — 提示词模板 + A/B 测试 + 插件系统
6. **progression-service** — 用户成长（等级 + 成就 + 排行榜）
7. **web-client** — Vue 3 前端 SPA
