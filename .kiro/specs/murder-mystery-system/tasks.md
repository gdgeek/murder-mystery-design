# 实现计划：剧本杀三系统平台

## 概述

基于已批准的需求和设计文档，将物料生成系统和游戏玩家系统拆分为增量式编码任务。剧本生成系统已实现，作为依赖引用。每个任务构建在前一个任务之上，最终完成完整系统集成。

## 任务

- [ ] 1. 初始化物料生成系统项目结构
  - [ ] 1.1 创建 `murder-mystery-material/` 项目骨架
    - 初始化 package.json（TypeScript + Express + Vitest + fast-check）
    - 创建 tsconfig.json
    - 创建 src/app.ts（Express 应用）和 src/server.ts（启动入口）
    - 创建 src/db/mysql.ts（MySQL 连接池）
    - 创建 src/db/migrations/ 目录和 SQL 迁移文件（material_jobs、sub_tasks 表）
    - 创建 storage/ 目录
    - _Requirements: 16.1, 19.3_

  - [ ] 1.2 实现 Script_JSON 校验器
    - 创建 src/validators/script-json.validator.ts
    - 校验顶层字段存在性（config、dmHandbook、playerHandbooks、materials、branchStructure）
    - 校验 playerHandbooks.length === config.playerCount
    - 校验 clue_card 的 clueId 与 dmHandbook.clueDistribution 一致
    - 返回结构化错误信息（字段名 + 错误描述）
    - _Requirements: 1.2, 1.3, 1.4, 2.1_

  - [ ]* 1.3 编写 Script_JSON 校验属性测试
    - **Property 1: Script_JSON 校验完整性**
    - **Validates: Requirements 1.2, 1.3, 1.4, 2.1**

  - [ ]* 1.4 编写无效 Script_JSON 错误响应属性测试
    - **Property 29: 无效 Script_JSON 返回结构化错误**
    - **Validates: Requirements 16.2**

- [ ] 2. 实现物料任务管理核心
  - [ ] 2.1 实现 MaterialJobService
    - 创建 src/services/material-job.service.ts
    - 实现 createJob：校验 Script_JSON → 创建 MaterialJob → 创建所有子任务（cover + player_pdf × N + dm_pdf + clue_card × M + ending_video × K）
    - 实现 getJob、getSubTasks、retrySubTask
    - 实现 executeJob：并发执行所有子任务，单个失败不影响其他
    - _Requirements: 2.2, 2.3, 2.4, 2.5_

  - [ ]* 2.2 编写子任务数量属性测试
    - **Property 2: MaterialJob 子任务数量正确性**
    - **Validates: Requirements 2.2**

  - [ ]* 2.3 编写创建-查询往返属性测试
    - **Property 3: MaterialJob 创建-查询往返一致性**
    - **Validates: Requirements 2.3**

  - [ ]* 2.4 编写故障隔离属性测试
    - **Property 4: 子任务故障隔离**
    - **Validates: Requirements 2.4, 6.4, 7.4**

  - [ ]* 2.5 编写重试保持不变属性测试
    - **Property 5: 重试保持已完成任务不变**
    - **Validates: Requirements 2.5**

- [ ] 3. 实现各物料生成器
  - [ ] 3.1 实现 AI 适配器
    - 创建 src/adapters/image-ai.adapter.ts（AI 生图接口 + 实现）
    - 创建 src/adapters/video-ai.adapter.ts（AI 生视频接口 + 实现）
    - 通过环境变量配置 API 密钥和端点
    - _Requirements: 19.4_

  - [ ] 3.2 实现 AssetStorage
    - 创建 src/services/asset-storage.ts
    - 实现 save（按 jobId 组织目录）、getFilePath、listFiles、createZip
    - _Requirements: 8.1, 8.2, 8.3_

  - [ ] 3.3 实现 CoverGenerator
    - 创建 src/services/cover-generator.ts
    - 从 Script_JSON 提取标题、时代、地点、主题组装提示词
    - 调用 ImageAIAdapter 生成图片
    - 存储到 AssetStorage
    - _Requirements: 3.1, 3.2, 3.3_

  - [ ] 3.4 实现 ScriptPdfGenerator
    - 创建 src/services/script-pdf-generator.ts
    - 实现 generatePlayerPdf：为每位玩家生成独立 PDF（角色名、背景、目标、关系、已知线索、行动指引）
    - 实现 generateDmPdf：生成 DM 手册 PDF（概述、角色列表、时间线、线索分发表、轮次指引、分支、结局、真相、判定规则）
    - 确保玩家 PDF 不包含其他角色 secrets
    - _Requirements: 4.1, 4.2, 4.3, 5.1, 5.2_

  - [ ] 3.5 实现 ClueCardGenerator
    - 创建 src/services/clue-card-generator.ts
    - 从线索卡 content 字段提取描述组装提示词
    - 调用 ImageAIAdapter，将配图与 clueId 关联存储
    - _Requirements: 6.1, 6.2, 6.3_

  - [ ] 3.6 实现 EndingVideoGenerator
    - 创建 src/services/ending-video-generator.ts
    - 从结局 narrative 字段提取叙述组装提示词
    - 调用 VideoAIAdapter，将视频与 endingId 关联存储
    - _Requirements: 7.1, 7.2, 7.3_

  - [ ]* 3.7 编写提示词构建属性测试
    - **Property 6: AI 提示词包含源数据关键字段**
    - **Validates: Requirements 3.1, 6.2, 7.2**

  - [ ]* 3.8 编写文件存储属性测试
    - **Property 7: 生成文件正确存储**
    - **Validates: Requirements 3.2, 4.4, 5.2, 6.3, 7.3, 8.1**

  - [ ]* 3.9 编写玩家 PDF 信息隔离属性测试
    - **Property 9: 玩家 PDF 信息隔离**
    - **Validates: Requirements 4.2**

  - [ ]* 3.10 编写 MaterialJob 序列化往返属性测试
    - **Property 12: MaterialJob 元数据序列化往返**
    - **Validates: Requirements 8.4**

- [ ] 4. 实现物料生成系统 REST API 和 UI
  - [ ] 4.1 实现 REST API 路由
    - 创建 src/routes/api.ts
    - POST /api/jobs — 提交 Script_JSON 创建任务
    - GET /api/jobs/:jobId — 查询任务状态
    - GET /api/jobs/:jobId/tasks — 查询子任务列表
    - POST /api/jobs/:jobId/tasks/:taskId/retry — 重试失败子任务
    - GET /api/jobs/:jobId/download — 下载 ZIP
    - GET /api/jobs/:jobId/files — 物料文件列表
    - GET /api/files/:filePath — 获取单个文件
    - GET /health — 健康检查
    - _Requirements: 16.1, 16.2, 16.3_

  - [ ] 4.2 实现内嵌 HTML UI
    - 创建 src/routes/ui/index.html（Bootstrap 5 CDN）
    - 创建 src/routes/ui/styles.css
    - 创建 src/routes/ui/app.js（上传 Script_JSON、任务列表、进度条、预览、下载、重试）
    - Express 静态文件服务配置
    - _Requirements: 16.1_

  - [ ] 4.3 创建 Dockerfile（多阶段构建）
    - _Requirements: 19.3_

- [ ] 5. 物料生成系统检查点
  - 确保所有测试通过，ask the user if questions arise.

- [ ] 6. 初始化游戏玩家系统项目结构
  - [ ] 6.1 创建 `murder-mystery-game/` 项目骨架
    - 初始化 package.json（TypeScript + Express + Socket.IO + Vitest + fast-check）
    - 创建 tsconfig.json
    - 创建 src/app.ts（Express + Socket.IO）和 src/server.ts
    - 创建 src/db/mysql.ts、src/db/redis.ts
    - 创建 src/db/migrations/（game_rooms、players、game_records、chat_messages 表）
    - _Requirements: 17.1, 19.3_

  - [ ] 6.2 实现房间码和二维码工具
    - 创建 src/utils/room-code.ts（6 位字母数字房间码生成）
    - 创建 src/utils/qr-code.ts（二维码生成）
    - _Requirements: 9.1_

  - [ ]* 6.3 编写房间码唯一性属性测试
    - **Property 13: 房间码唯一性**
    - **Validates: Requirements 9.1**

- [ ] 7. 实现房间管理服务
  - [ ] 7.1 实现 RoomService
    - 创建 src/services/room.service.ts
    - 实现 createRoom：创建房间 + 生成 Room_Code + 创建者作为首个玩家
    - 实现 joinRoom：检查人数上限 + 添加玩家
    - 实现 leaveRoom、getRoom、updateStatus
    - _Requirements: 9.1, 9.2, 9.3, 10.3, 10.5_

  - [ ] 7.2 实现 PlayerService
    - 创建 src/services/player.service.ts
    - 实现 selectCharacter（独占分配）、setReady、updateConnection
    - 实现 getDisconnectedPlayers（超时检测）
    - _Requirements: 11.2, 11.3, 15.3_

  - [ ]* 7.3 编写房间创建初始状态属性测试
    - **Property 14: 房间创建初始状态正确**
    - **Validates: Requirements 9.2, 9.3**

  - [ ]* 7.4 编写房间人数上限属性测试
    - **Property 15: 房间人数上限强制执行**
    - **Validates: Requirements 10.3**

  - [ ]* 7.5 编写角色独占分配属性测试
    - **Property 18: 角色独占分配**
    - **Validates: Requirements 11.2, 11.3**

- [ ] 8. 实现游戏流程控制
  - [ ] 8.1 实现 GameFlowService
    - 创建 src/services/game-flow.service.ts
    - 实现状态机：waiting → selecting → playing → ended
    - 实现 startGame、advancePhase（reading → investigation → discussion）
    - 实现 distributeClues（按 DM 手册线索分发表分发）
    - 实现 initiateVote、submitVote、determineEnding
    - Redis 存储实时游戏状态
    - _Requirements: 11.1, 11.4, 11.5, 12.2, 12.3, 12.4, 12.5, 12.6, 14.1_

  - [ ]* 8.2 编写房间状态机属性测试
    - **Property 17: 人数满员自动进入选角**
    - **Property 20: 游戏开始状态转换**
    - **Property 21: 游戏阶段顺序**
    - **Validates: Requirements 11.1, 11.5, 12.2**

  - [ ]* 8.3 编写线索分发正确性属性测试
    - **Property 22: 线索分发正确性**
    - **Validates: Requirements 12.3, 12.4**

  - [ ]* 8.4 编写投票决定分支属性测试
    - **Property 23: 投票决定分支走向**
    - **Property 25: 分支路径决定最终结局**
    - **Validates: Requirements 12.6, 14.1**

- [ ] 9. 实现 AI 主持人和聊天
  - [ ] 9.1 实现 AIDMService
    - 创建 src/services/ai-dm.service.ts
    - 创建 src/adapters/llm.adapter.ts（LLM 适配器）
    - 实现 generateOpening、generateRoundNarration、generateBranchResult
    - 实现 generatePlayerEvaluation、answerQuestion
    - LLM 失败时回退到 DM 手册预设文本
    - _Requirements: 12.1, 12.7, 13.3, 14.5_

  - [ ] 9.2 实现 ChatService
    - 创建 src/services/chat.service.ts
    - 实现 sendMessage（包含昵称和时间戳）、getMessages
    - _Requirements: 13.1, 13.2_

  - [ ]* 9.3 编写聊天消息格式属性测试
    - **Property 24: 聊天消息格式完整**
    - **Validates: Requirements 13.2**

  - [ ]* 9.4 编写游戏结束状态属性测试
    - **Property 26: 游戏结束状态完整性**
    - **Validates: Requirements 14.5, 14.6**

- [ ] 10. 实现 WebSocket 事件处理
  - [ ] 10.1 实现 WebSocket 事件处理器
    - 创建 src/websocket/events.ts（事件类型定义）
    - 创建 src/websocket/handler.ts
    - 处理客户端事件：room:join、room:leave、character:select、player:ready、chat:send、vote:submit、dm:ask
    - 推送服务端事件：room:playerJoined/Left、character:updated、game:started/phaseChanged/clueReceived/dmMessage/ended、chat:message、vote:initiated/result
    - _Requirements: 17.2, 17.3_

  - [ ] 10.2 实现房间生命周期管理
    - 创建 src/services/room-lifecycle.service.ts
    - 实现定时清理：waiting 超 30 分钟无玩家 → 关闭；ended 超 24 小时 → 清理 Redis
    - 实现断线处理：保留位置 5 分钟，超时 AI 接管
    - _Requirements: 15.1, 15.2, 15.3, 15.4_

  - [ ]* 10.3 编写房间生命周期属性测试
    - **Property 27: 房间生命周期清理**
    - **Validates: Requirements 15.1, 15.2**

  - [ ]* 10.4 编写玩家断线处理属性测试
    - **Property 28: 玩家断线处理**
    - **Validates: Requirements 15.3, 15.4**

- [ ] 11. 实现游戏玩家系统 REST API 和 UI
  - [ ] 11.1 实现 REST API 路由
    - 创建 src/routes/api.ts
    - POST /api/rooms — 创建房间
    - GET /api/rooms/:roomCode — 查询房间信息
    - GET /api/rooms/:roomCode/qrcode — 获取二维码
    - GET /api/scripts — 获取可用剧本列表
    - GET /health — 健康检查
    - _Requirements: 17.1_

  - [ ] 11.2 实现内嵌 HTML UI 页面
    - 创建 src/routes/ui/index.html（首页：创建房间/输入房间码）
    - 创建 src/routes/ui/room.html（房间页：二维码、玩家列表、选角）
    - 创建 src/routes/ui/game.html（游戏页：聊天、线索、投票、AI_DM、结局）
    - 创建 src/routes/ui/styles.css 和 src/routes/ui/app.js
    - 使用 Bootstrap 5 CDN + Socket.IO 客户端
    - _Requirements: 9.4, 14.2, 14.3, 14.4_

  - [ ] 11.3 创建 Dockerfile（多阶段构建）
    - _Requirements: 19.3_

- [ ] 12. 游戏玩家系统检查点
  - 确保所有测试通过，ask the user if questions arise.

- [ ] 13. Docker Compose 集成
  - [ ] 13.1 创建 docker-compose.yml
    - 配置 generator、murder-mystery-material、murder-mystery-game、mysql、redis 五个服务
    - 创建 init-databases.sql（初始化 material_db 和 game_db）
    - 配置环境变量（AI 服务密钥通过 .env 文件注入）
    - 配置 volumes（mysql-data、material-storage）
    - _Requirements: 19.1, 19.2, 19.4_

  - [ ] 13.2 创建 .env.example
    - 列出所有环境变量及说明
    - _Requirements: 19.4_

- [ ] 14. 最终检查点
  - 确保所有测试通过，ask the user if questions arise.

## 备注

- 标记 `*` 的任务为可选测试任务，可跳过以加速 MVP 开发
- 每个任务引用具体需求编号以确保可追溯性
- 属性测试验证通用正确性属性，单元测试验证具体示例和边界情况
- 检查点确保增量验证
- 剧本生成系统已实现，通过 `@gdgeek/murder-mystery-shared` 包和 REST API 集成
