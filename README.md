# 剧本杀系统 — 设计知识库

本工程是剧本杀平台的统一设计知识库，不包含任何实现代码。平台由三个独立系统组成，各系统在独立的代码仓库中实现，通过引用本知识库（git submodule）进行协同设计与开发。

## 设计原则

**KISS（Keep It Simple, Stupid）**。不设计知识库、设计师身份、技能牌、经济系统、A/B 测试、排行榜、成就收藏、插件系统、国际化。专注核心功能，先做中文版。

## 三个系统

| 系统 | 端口 | 职责 | 仓库 | 状态 |
|------|------|------|------|------|
| 剧本生成系统 | 3002 | AI 生成完整剧本杀 JSON（DM手册、玩家手册、物料、分支结构、多结局） | [murder-mystery-generator](https://github.com/gdgeek/murder-mystery-generator) | ✅ 已实现 |
| 物料生成系统 | 3003 | 读取剧本 JSON，生成封面（AI生图）、玩家剧本PDF、DM手册PDF、线索卡（AI生图）、结局视频（AI生视频） | murder-mystery-material（待创建） | 🔲 待开发 |
| 游戏玩家系统 | 3004 | Web 游玩平台：创建房间、扫码加入、选角、AI主持人自动主持全流程 | murder-mystery-game（待创建） | 🔲 待开发 |

### 系统关系

```
剧本生成系统 ──Script_JSON──► 物料生成系统（生成可视化物料）
剧本生成系统 ──Script_JSON──► 游戏玩家系统（加载剧本开始游戏）
```

三个系统通过 Script_JSON 数据格式连接，各自独立部署，通过 Docker Compose 统一编排。

## 技术栈

- 全栈 TypeScript（Node.js + Express）
- 内嵌 HTML UI（Bootstrap 5 CDN + 原生 JS），不使用前端框架
- MySQL 8.0 + Redis
- Socket.IO（游戏系统实时通信）
- Docker + Docker Compose
- Vitest + fast-check（测试）
- 共享类型包：`@gdgeek/murder-mystery-shared`

## 知识库结构

```
.kiro/
├── specs/
│   └── murder-mystery-system/
│       ├── requirements.md    # 需求文档（18项需求）
│       ├── design.md          # 设计文档（架构、接口、数据模型、API、WebSocket、正确性属性）
│       └── tasks.md           # 实现任务清单（14个顶层任务）
├── work-log/                  # 工作日志
└── skills/                    # 剧本杀创作领域知识
init-db/                       # 数据库初始化脚本
.env.example                   # 环境变量模板
```

## 如何使用

各子项目通过 git submodule 引用本知识库：

```bash
# 在子项目中添加知识库
git submodule add git@github.com:gdgeek/murder-mystery-design.git design-kb

# 查看设计文档
cat design-kb/.kiro/specs/murder-mystery-system/design.md

# 查看任务清单
cat design-kb/.kiro/specs/murder-mystery-system/tasks.md
```

## 快速部署（全部系统）

```bash
cp .env.example .env
# 编辑 .env 填入 AI 服务 API 密钥
docker compose up
```
