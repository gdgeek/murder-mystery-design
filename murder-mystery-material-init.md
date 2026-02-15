# murder-mystery-material 项目初始化指南

## 项目简介

剧本杀物料生成系统。读取剧本生成系统输出的 Script_JSON，调用 AI 服务生成可视化物料：封面图片、玩家剧本 PDF、DM 手册 PDF、线索卡图片、结局视频。

属于剧本杀三系统平台的第二个系统，设计文档位于 [murder-mystery-design](https://github.com/gdgeek/murder-mystery-design) 知识库。

## 技术栈

- TypeScript + Node.js + Express
- MySQL 8.0（数据库名：`material_db`）
- 内嵌 HTML UI（Bootstrap 5 CDN + 原生 JS），不使用前端框架
- Vitest + fast-check（测试）
- `@gdgeek/murder-mystery-shared`（共享类型包）
- Docker

## 项目结构

```
murder-mystery-material/
├── src/
│   ├── server.ts                      # Express 启动入口（端口 3003）
│   ├── app.ts                         # Express 应用配置
│   ├── db/
│   │   ├── mysql.ts                   # MySQL 连接池
│   │   └── migrations/                # SQL 迁移文件
│   │       └── 001-init.sql
│   ├── services/
│   │   ├── material-job.service.ts    # 任务管理核心
│   │   ├── cover-generator.ts         # 封面生成
│   │   ├── script-pdf-generator.ts    # PDF 生成
│   │   ├── clue-card-generator.ts     # 线索卡生成
│   │   ├── ending-video-generator.ts  # 结局视频生成
│   │   └── asset-storage.ts           # 文件存储
│   ├── adapters/
│   │   ├── image-ai.adapter.ts        # AI 生图适配器
│   │   └── video-ai.adapter.ts        # AI 生视频适配器
│   ├── routes/
│   │   ├── api.ts                     # REST API 路由
│   │   └── ui/                        # 内嵌 HTML UI
│   │       ├── index.html
│   │       ├── styles.css
│   │       └── app.js
│   └── validators/
│       └── script-json.validator.ts   # Script_JSON 校验
├── tests/
│   ├── services/
│   ├── validators/
│   ├── properties/                    # 属性测试（fast-check）
│   └── routes/
├── storage/                           # 本地文件存储（gitignore）
├── design-kb/                         # git submodule → murder-mystery-design
├── .env.example
├── .gitignore
├── Dockerfile
├── package.json
├── tsconfig.json
└── README.md
```

## 快速开始

```bash
# 1. 克隆项目
git clone https://github.com/gdgeek/murder-mystery-material.git
cd murder-mystery-material

# 2. 添加设计知识库 submodule
git submodule add git@github.com:gdgeek/murder-mystery-design.git design-kb

# 3. 安装依赖
npm install

# 4. 配置环境变量
cp .env.example .env
# 编辑 .env 填入数据库和 AI 服务配置

# 5. 启动开发
npm run dev
```

## 环境变量（.env.example）

```env
# 服务端口
PORT=3003

# MySQL
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=
MYSQL_DATABASE=material_db

# AI 生图服务
IMAGE_AI_API_KEY=
IMAGE_AI_ENDPOINT=

# AI 生视频服务
VIDEO_AI_API_KEY=
VIDEO_AI_ENDPOINT=
```

## package.json 关键依赖

```json
{
  "name": "murder-mystery-material",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js",
    "test": "vitest run",
    "test:watch": "vitest"
  },
  "dependencies": {
    "express": "^4.18.0",
    "mysql2": "^3.9.0",
    "uuid": "^9.0.0",
    "archiver": "^7.0.0",
    "pdfkit": "^0.15.0",
    "@gdgeek/murder-mystery-shared": "latest"
  },
  "devDependencies": {
    "typescript": "^5.4.0",
    "tsx": "^4.7.0",
    "@types/express": "^4.17.0",
    "@types/node": "^20.0.0",
    "@types/uuid": "^9.0.0",
    "@types/archiver": "^6.0.0",
    "vitest": "^1.6.0",
    "fast-check": "^3.19.0"
  }
}
```

## tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

## 数据库迁移（001-init.sql）

```sql
CREATE TABLE IF NOT EXISTS material_jobs (
  id VARCHAR(36) PRIMARY KEY,
  script_json_id VARCHAR(36) NOT NULL,
  script_json_data JSON NOT NULL,
  status ENUM('pending', 'processing', 'completed', 'failed') DEFAULT 'pending',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_status (status)
);

CREATE TABLE IF NOT EXISTS sub_tasks (
  id VARCHAR(36) PRIMARY KEY,
  job_id VARCHAR(36) NOT NULL,
  type ENUM('cover', 'player_pdf', 'dm_pdf', 'clue_card', 'ending_video') NOT NULL,
  reference_id VARCHAR(100) NOT NULL,
  status ENUM('pending', 'processing', 'completed', 'failed') DEFAULT 'pending',
  result_path VARCHAR(500) NULL,
  error_message TEXT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (job_id) REFERENCES material_jobs(id),
  INDEX idx_job_status (job_id, status)
);
```

## REST API

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/jobs` | 提交 Script_JSON 创建物料任务 |
| GET | `/api/jobs/:jobId` | 查询任务状态 |
| GET | `/api/jobs/:jobId/tasks` | 查询子任务列表 |
| POST | `/api/jobs/:jobId/tasks/:taskId/retry` | 重试失败子任务 |
| GET | `/api/jobs/:jobId/download` | 下载物料 ZIP |
| GET | `/api/jobs/:jobId/files` | 物料文件列表 |
| GET | `/api/files/:filePath` | 获取单个资源文件 |
| GET | `/health` | 健康检查 |

## .gitignore

```
node_modules/
dist/
storage/
.env
*.log
```

## Dockerfile

```dockerfile
# 构建阶段
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY tsconfig.json ./
COPY src/ ./src/
RUN npm run build

# 运行阶段
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY --from=builder /app/dist ./dist
COPY src/routes/ui ./dist/routes/ui
RUN mkdir -p storage
EXPOSE 3003
CMD ["node", "dist/server.js"]
```

## 实现顺序

参照设计知识库 `tasks.md` 中的任务 1-5：

1. 项目骨架（server.ts、app.ts、mysql.ts、迁移文件）
2. Script_JSON 校验器
3. MaterialJobService（任务管理核心）
4. AI 适配器 + AssetStorage + 各生成器
5. REST API + 内嵌 HTML UI + Dockerfile

详细任务描述见 `design-kb/.kiro/specs/murder-mystery-system/tasks.md`
