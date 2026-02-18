# 需求文档：剧本杀三系统平台

## 简介

本平台由三个独立但协作的系统组成，围绕剧本杀游戏的完整生命周期：

1. **剧本生成系统**（已实现）：通过 AI 生成完整的剧本杀 JSON 数据，包含 DM 手册、玩家手册、物料定义、分支结构和多结局。
2. **物料生成系统**（新建）：读取剧本 JSON，调用 AI 生图/生视频服务，生成可视化物料（封面、玩家剧本 PDF、DM 手册 PDF、线索卡图片、结局视频）。
3. **游戏玩家系统**（新建）：提供 Web 游玩平台，支持创建房间、扫码加入、选角、AI 主持人自动主持游戏全流程。

设计原则：KISS（Keep It Simple, Stupid）。不设计知识库、设计师身份、技能牌、经济系统、A/B 测试、排行榜、成就收藏、插件系统、国际化。专注核心功能，先做中文版。

## 技术选型

- **语言**：全栈 TypeScript
- **后端**：Node.js + Express
- **前端**：内嵌 HTML UI（Bootstrap 5 CDN + 原生 JS），不使用前端框架
- **数据库**：MySQL 8.0 + Redis
- **实时通信**：WebSocket（Socket.IO）
- **容器化**：Docker + Docker Compose
- **测试**：Vitest + fast-check

## 术语表

- **Script_JSON**：剧本生成系统输出的 JSON 数据，包含完整剧本内容（DM 手册、玩家手册、物料、分支结构、多结局）
- **Material_Service（物料服务）**：物料生成系统的核心服务，负责读取 Script_JSON 并调度各类物料生成任务
- **Cover_Generator（封面生成器）**：调用 AI 生图服务，根据剧本主题和风格生成封面图片
- **Script_PDF_Generator（剧本PDF生成器）**：将玩家手册和 DM 手册渲染为可阅读的 PDF 文档
- **Clue_Card_Generator（线索卡生成器）**：调用 AI 生图服务，为每张线索卡生成配图
- **Ending_Video_Generator（结局视频生成器）**：调用 AI 生视频服务，为结局生成短视频
- **Asset_Storage（资源存储）**：统一的文件存储层，管理所有生成的图片、PDF、视频文件，默认本地存储
- **Game_Room（游戏房间）**：一次剧本杀游戏实例，包含房间码、玩家列表、游戏状态
- **AI_DM（AI主持人）**：基于 LLM 的自动化主持人，根据 DM 手册主持游戏全流程
- **Player（玩家）**：通过扫码或输入房间码加入游戏的用户，无需注册账户
- **Room_Code（房间码）**：6位数字/字母组合，用于标识和加入游戏房间
- **QR_Code（二维码）**：包含房间加入链接的二维码，供玩家扫码加入

## 需求

### 需求 1：剧本生成系统（已实现，作为输入）

**用户故事：** 作为剧本创作者，我希望通过配置参数让 AI 生成完整的剧本杀 JSON 数据，以便后续系统使用。

#### 验收标准

1. THE Script_JSON SHALL 包含以下顶层结构：剧本元信息（标题、类型、玩家人数、时长）、DM 手册（DMHandbook）、玩家手册数组（PlayerHandbook[]）、物料数组（Material[]）、分支结构（BranchStructure）
2. WHEN 物料生成系统或游戏玩家系统读取 Script_JSON 时，THE Script_JSON SHALL 通过 `@gdgeek/murder-mystery-shared` 包提供的 TypeScript 类型定义进行结构校验
3. THE Script_JSON SHALL 确保玩家手册数量与剧本元信息中的玩家人数一致
4. THE Script_JSON SHALL 确保物料中每张线索卡的 clueId 与 DM 手册线索分发表中的引用一致


### 需求 2：物料生成任务管理

**用户故事：** 作为剧本创作者，我希望上传或选择一个已生成的剧本 JSON，系统自动创建物料生成任务，以便我能追踪各项物料的生成进度。

#### 验收标准

1. WHEN 用户提交一个 Script_JSON 时，THE Material_Service SHALL 校验 JSON 结构完整性（包含 DMHandbook、PlayerHandbook[]、Material[]、BranchStructure），校验失败返回具体错误字段
2. WHEN Script_JSON 校验通过后，THE Material_Service SHALL 创建一个物料生成任务（MaterialJob），包含以下子任务：封面生成、每位玩家的剧本 PDF 生成、DM 手册 PDF 生成、每张线索卡的图片生成、每个结局的视频生成
3. WHEN MaterialJob 创建后，THE Material_Service SHALL 返回任务 ID，用户可通过任务 ID 查询各子任务的状态（pending/processing/completed/failed）
4. WHEN 任一子任务失败时，THE Material_Service SHALL 记录错误信息，其他子任务继续执行不受影响
5. WHEN 用户请求重试失败的子任务时，THE Material_Service SHALL 仅重新执行该失败子任务，已完成的子任务保持不变

### 需求 3：封面图片生成

**用户故事：** 作为剧本创作者，我希望系统根据剧本的主题和风格自动生成一张封面图片，以便用于剧本展示。

#### 验收标准

1. WHEN 封面生成子任务执行时，THE Cover_Generator SHALL 从 Script_JSON 中提取标题、时代背景、地点设定、主题风格，组装为 AI 生图提示词
2. WHEN Cover_Generator 调用 AI 生图服务成功后，THE Cover_Generator SHALL 将生成的图片存储到 Asset_Storage，并将存储路径关联到 MaterialJob
3. IF AI 生图服务调用失败，THEN THE Cover_Generator SHALL 将子任务标记为 failed 并记录错误信息

### 需求 4：玩家剧本 PDF 生成

**用户故事：** 作为玩家，我希望获得一份排版精美的角色剧本 PDF，以便在游戏中阅读自己的角色信息。

#### 验收标准

1. WHEN 玩家剧本 PDF 子任务执行时，THE Script_PDF_Generator SHALL 为每位玩家生成独立的 PDF 文件，包含：角色名称、角色背景故事、角色目标、角色关系、已知线索、各轮次行动指引
2. THE Script_PDF_Generator SHALL 确保每份玩家 PDF 仅包含该角色应知的信息，不包含其他角色的秘密信息
3. WHEN 生成的玩家 PDF 数量与 Script_JSON 中的玩家手册数量不一致时，THE Script_PDF_Generator SHALL 将子任务标记为 failed 并报告数量不匹配错误
4. THE Script_PDF_Generator SHALL 将生成的 PDF 文件存储到 Asset_Storage

### 需求 5：DM 手册 PDF 生成

**用户故事：** 作为 DM（主持人），我希望获得一份完整的 DM 手册 PDF，以便参考游戏的完整信息。

#### 验收标准

1. WHEN DM 手册 PDF 子任务执行时，THE Script_PDF_Generator SHALL 生成一份 DM 手册 PDF，包含：游戏概述、角色列表、完整时间线、线索分发表、各轮次流程指引、分支决策点、多结局描述、真相还原、结局判定规则
2. THE Script_PDF_Generator SHALL 将 DM 手册 PDF 存储到 Asset_Storage

### 需求 6：线索卡图片生成

**用户故事：** 作为剧本创作者，我希望系统为每张线索卡生成配图，以便在游戏中展示更具沉浸感的线索。

#### 验收标准

1. WHEN 线索卡图片子任务执行时，THE Clue_Card_Generator SHALL 为 Script_JSON 中每张类型为 clue_card 的物料生成一张配图
2. WHEN Clue_Card_Generator 生成配图时，THE Clue_Card_Generator SHALL 从线索卡的 content 字段提取描述信息，组装为 AI 生图提示词
3. THE Clue_Card_Generator SHALL 将每张配图与对应线索卡的 clueId 关联存储到 Asset_Storage
4. IF 某张线索卡的图片生成失败，THEN THE Clue_Card_Generator SHALL 将该线索卡子任务标记为 failed，其他线索卡继续生成

### 需求 7：结局视频生成

**用户故事：** 作为剧本创作者，我希望系统为每个结局生成一段短视频，以便在游戏结束时播放增强沉浸感。

#### 验收标准

1. WHEN 结局视频子任务执行时，THE Ending_Video_Generator SHALL 为 Script_JSON 中 BranchStructure.endings 的每个结局生成一段 10-30 秒的短视频
2. WHEN Ending_Video_Generator 生成视频时，THE Ending_Video_Generator SHALL 从结局的 narrative 字段提取叙述内容，组装为 AI 生视频提示词
3. THE Ending_Video_Generator SHALL 将每段视频与对应结局的 endingId 关联存储到 Asset_Storage
4. IF 某个结局的视频生成失败，THEN THE Ending_Video_Generator SHALL 将该子任务标记为 failed，其他结局视频继续生成

### 需求 8：资源存储与下载

**用户故事：** 作为剧本创作者，我希望能查看和下载所有已生成的物料文件，以便在需要时使用。

#### 验收标准

1. THE Asset_Storage SHALL 以本地文件系统存储所有生成的资源文件（图片、PDF、视频），按 MaterialJob ID 组织目录结构
2. WHEN 用户请求下载某个 MaterialJob 的全部物料时，THE Asset_Storage SHALL 将所有已完成的资源文件打包为 ZIP 文件返回
3. WHEN 用户请求查看某个 MaterialJob 的物料列表时，THE Asset_Storage SHALL 返回每个资源文件的类型、文件名、大小、生成状态和访问 URL
4. WHEN 序列化 MaterialJob 元数据为 JSON 后再反序列化时，THE Material_Service SHALL 产生与原始数据等价的结果


### 需求 9：创建游戏房间

**用户故事：** 作为玩家，我希望选择一个剧本并创建游戏房间，以便邀请其他玩家一起游玩。

#### 验收标准

1. WHEN 用户选择一个剧本并点击"创建房间"时，THE Game_Room SHALL 创建一个新房间，生成唯一的 6 位 Room_Code 和对应的 QR_Code
2. WHEN Game_Room 创建时，THE Game_Room SHALL 将房间状态设为 waiting，记录关联的 Script_JSON ID 和创建者昵称
3. THE Game_Room SHALL 在创建时为创建者分配一个临时玩家身份（昵称），无需注册账户
4. WHEN 房间创建成功后，THE Game_Room SHALL 展示房间页面，包含 Room_Code、QR_Code 和当前已加入的玩家列表

### 需求 10：加入游戏房间

**用户故事：** 作为玩家，我希望通过扫码或输入房间码加入游戏房间，以便参与剧本杀游戏。

#### 验收标准

1. WHEN 玩家扫描 QR_Code 或在浏览器中输入房间码时，THE Game_Room SHALL 在玩家设备上打开游戏页面
2. WHEN 玩家进入游戏页面后，THE Game_Room SHALL 要求玩家输入昵称，然后加入房间
3. WHEN 玩家加入房间时，THE Game_Room SHALL 检查当前玩家人数是否已达到剧本要求的上限，已满则拒绝加入并提示"房间已满"
4. WHEN 玩家成功加入房间后，THE Game_Room SHALL 通过 WebSocket 实时通知房间内所有玩家更新玩家列表
5. WHILE Game_Room 处于 waiting 状态时，THE Game_Room SHALL 允许玩家自由加入和离开

### 需求 11：角色选择

**用户故事：** 作为玩家，我希望在游戏开始前选择自己想扮演的角色，以便获得对应的角色剧本。

#### 验收标准

1. WHEN 房间内玩家人数达到剧本要求时，THE Game_Room SHALL 自动进入选角阶段（selecting 状态），展示所有可选角色的名称和简介
2. WHEN 玩家选择一个角色时，THE Game_Room SHALL 将该角色标记为已选，通过 WebSocket 通知所有玩家更新角色选择状态
3. WHEN 某角色已被其他玩家选择时，THE Game_Room SHALL 将该角色显示为不可选状态
4. WHEN 所有玩家都完成角色选择后，THE Game_Room SHALL 显示"开始游戏"按钮
5. WHEN 所有玩家点击"开始游戏"后，THE Game_Room SHALL 将房间状态切换为 playing，通知 AI_DM 开始主持

### 需求 12：AI 主持人游戏流程

**用户故事：** 作为玩家，我希望 AI 主持人自动主持游戏全流程，以便无需人类 DM 即可开始游戏。

#### 验收标准

1. WHEN Game_Room 进入 playing 状态时，THE AI_DM SHALL 根据 DM 手册中的游戏概述向所有玩家发送开场白
2. WHEN AI_DM 推进游戏轮次时，THE AI_DM SHALL 按照 DM 手册中的轮次流程指引，依次推进阅读、搜证、推证三个阶段
3. WHEN 进入搜证阶段时，THE AI_DM SHALL 根据 DM 手册中的线索分发表，向指定玩家分发对应的线索卡
4. WHEN AI_DM 分发线索时，THE AI_DM SHALL 确保每张线索卡仅发送给 DM 手册中指定的目标角色对应的玩家
5. WHEN 到达分支决策点时，THE AI_DM SHALL 向所有玩家展示投票问题和选项，收集投票结果
6. WHEN 投票完成后，THE AI_DM SHALL 根据投票结果选择对应的剧情分支，向所有玩家播报分支结果叙述
7. IF AI_DM 调用 LLM 失败，THEN THE AI_DM SHALL 回退到使用 DM 手册中的预设文本继续主持，不中断游戏

### 需求 13：游戏内实时交流

**用户故事：** 作为玩家，我希望在游戏过程中能与其他玩家实时文字交流，以便进行推理讨论。

#### 验收标准

1. WHILE Game_Room 处于 playing 状态时，THE Game_Room SHALL 提供公共聊天频道，所有玩家可发送和接收文字消息
2. WHEN 玩家发送聊天消息时，THE Game_Room SHALL 通过 WebSocket 实时广播给房间内所有玩家，消息包含发送者昵称和时间戳
3. WHEN 玩家向 AI_DM 提问时，THE AI_DM SHALL 根据 DM 手册中的信息范围回答问题，避免泄露不应公开的信息

### 需求 14：游戏结局与结算

**用户故事：** 作为玩家，我希望在游戏结束时看到结局揭示和个人评价，以便了解游戏结果。

#### 验收标准

1. WHEN 游戏进入最终阶段时，THE AI_DM SHALL 根据游戏过程中所有投票结果和分支路径，确定最终结局
2. WHEN 结局确定后，THE Game_Room SHALL 向所有玩家展示结局叙述、真相还原和每位玩家的个人结局
3. WHEN 结局展示时，IF 该结局有关联的视频资源，THEN THE Game_Room SHALL 向所有玩家同步播放结局视频
4. IF 结局视频不存在或播放失败，THEN THE Game_Room SHALL 以文字形式展示结局叙述，不影响游戏结算
5. WHEN 游戏结束后，THE AI_DM SHALL 为每位玩家生成简短的个人评价，包含推理表现和角色扮演表现的文字点评
6. WHEN 游戏结束后，THE Game_Room SHALL 将房间状态设为 ended，记录游戏结果（选择的分支路径、最终结局、玩家评价）

### 需求 15：游戏房间生命周期管理

**用户故事：** 作为系统管理员，我希望游戏房间有合理的生命周期管理，以便释放服务器资源。

#### 验收标准

1. WHEN Game_Room 处于 waiting 状态超过 30 分钟且无玩家加入时，THE Game_Room SHALL 自动关闭房间并释放资源
2. WHEN Game_Room 处于 ended 状态超过 24 小时后，THE Game_Room SHALL 清理 WebSocket 连接和 Redis 中的实时状态数据，数据库中的游戏记录保留
3. WHEN 玩家在游戏过程中断开连接时，THE Game_Room SHALL 保留该玩家的位置 5 分钟，期间玩家可重新连接恢复游戏状态
4. IF 玩家断开超过 5 分钟未重连，THEN THE AI_DM SHALL 自动接管该玩家角色，以最基本的方式参与游戏（自动投票、不主动发言）

### 需求 16：物料生成系统 REST API

**用户故事：** 作为前端开发者，我希望通过 REST API 访问物料生成功能，以便在前端界面中集成物料管理。

#### 验收标准

1. THE Material_Service SHALL 提供以下 REST API 端点：提交 Script_JSON 创建 MaterialJob（POST）、查询 MaterialJob 状态（GET）、查询子任务列表及状态（GET）、重试失败子任务（POST）、下载物料 ZIP（GET）、获取单个资源文件（GET）
2. WHEN API 接收到无效的 Script_JSON 时，THE Material_Service SHALL 返回包含具体错误字段和错误描述的结构化错误响应
3. WHEN API 接收到不存在的 MaterialJob ID 时，THE Material_Service SHALL 返回 404 错误

### 需求 17：游戏玩家系统 REST API 与 WebSocket

**用户故事：** 作为前端开发者，我希望通过 REST API 和 WebSocket 访问游戏功能，以便构建游戏前端界面。

#### 验收标准

1. THE Game_Room SHALL 提供以下 REST API 端点：创建房间（POST）、查询房间信息（GET）、获取房间二维码（GET）、获取可用剧本列表（GET）
2. THE Game_Room SHALL 提供 WebSocket 连接端点，支持以下事件：加入房间、离开房间、选择角色、准备就绪、发送聊天消息、投票、向 AI_DM 提问
3. THE Game_Room SHALL 通过 WebSocket 向客户端推送以下事件：玩家加入/离开、角色选择更新、游戏开始、轮次更新、线索分发、聊天消息、投票发起/结果、AI_DM 发言、游戏结束

### 需求 18：可游玩结构（Playable Structure）

**用户故事：** 作为玩家和DM，我希望剧本内容按"幕"组织为线性可游玩序列（序幕→多个中间幕→终幕），每幕包含故事叙述、搜证目标、交流建议和投票/决策，以便能按顺序体验完整游戏。

#### 验收标准

1. THE Script SHALL 支持可选的 `playableStructure` 字段，包含序幕（Prologue）、多个中间幕（Act[]）和终幕（Finale）的有序结构
2. WHEN 定义 Act 时，THE Act SHALL 包含幕索引、标题、故事叙述、搜证目标、线索ID列表、交流建议和投票定义
3. WHEN 生成 PlayableStructure 时，THE Generator SHALL 确保中间幕数量等于 `config.roundStructure.totalRounds`
4. THE PlayableStructure SHALL 包含双视角幕结构：DM可游玩手册（按幕的主持指引）和玩家可游玩手册（按幕的角色内容）
5. WHEN 系统读取旧版 Script（无 playableStructure 字段）时，THE System SHALL 正常运行不报错
6. WHEN 提供迁移工具时，THE MigrationTool SHALL 将旧版 roundGuides/roundActions 映射为幕结构，且不修改原始数据
7. WHEN Generator 生成 PlayableStructure 后，THE Generator SHALL 校验幕数量一致性、线索双向一致性和线索分发指令一致性

### 需求 19：容器化部署

**用户故事：** 作为开发者，我希望三个系统都能通过 Docker Compose 一键部署，以便快速搭建开发和生产环境。

#### 验收标准

1. THE System SHALL 提供 docker-compose.yml 文件，包含以下服务：剧本生成系统、物料生成系统、游戏玩家系统、MySQL、Redis
2. WHEN 执行 docker-compose up 命令时，THE System SHALL 启动所有服务并使系统可访问
3. THE System SHALL 为物料生成系统和游戏玩家系统分别提供 Dockerfile，支持多阶段构建
4. THE System SHALL 通过环境变量配置 AI 服务的 API 密钥和端点地址，不硬编码在代码中
