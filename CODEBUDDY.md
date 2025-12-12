# CODEBUDDY.md 本文件为 CodeBuddy 在此代码仓库中工作时提供指导。

## 项目概述

New API 是一个基于 [One API](https://github.com/songquanpeng/one-api) 构建的大模型网关与 AI 资产管理系统。它为多个 AI 服务商（OpenAI、Claude、Gemini 等）提供统一的 API 接口，具备渠道管理、配额/计费、身份认证和速率限制等功能。

## 构建与开发命令

### 后端 (Go)
```bash
# 运行后端开发服务器
go run main.go

# 构建带版本号的二进制文件
go build -ldflags "-s -w -X 'github.com/QuantumNous/new-api/common.Version=$(cat VERSION)'" -o new-api
```

### 前端 (React + Vite + Bun)
```bash
cd web

# 安装依赖
bun install

# 开发服务器
bun run dev

# 生产环境构建
DISABLE_ESLINT_PLUGIN='true' VITE_REACT_APP_VERSION=$(cat ../VERSION) bun run build

# 代码检查
bun run lint        # 使用 prettier 检查
bun run lint:fix    # 使用 prettier 修复
bun run eslint      # ESLint 检查
bun run eslint:fix  # ESLint 修复
```

### 完整构建 (Makefile)
```bash
make all             # 先构建前端，再启动后端
make build-frontend  # 仅构建前端
make start-backend   # 仅启动后端
```

### Docker
```bash
docker-compose up -d              # 启动服务（含 PostgreSQL + Redis）
docker build -t new-api .         # 构建镜像
```

## 架构

### 后端结构

```
├── main.go              # 入口文件，初始化数据库/Redis/路由，启动 Gin 服务器
├── router/              # 路由定义
│   ├── main.go          # SetRouter - 组合所有路由
│   ├── api-router.go    # /api/* 路由（管理员、用户、渠道、令牌等）
│   └── relay-router.go  # /v1/* 路由（OpenAI 兼容的 API 端点）
├── controller/          # /api/* 路由的 HTTP 处理器
├── relay/               # 代理 AI 请求的核心中继逻辑
│   ├── *_handler.go     # 按类型划分的请求处理器（聊天、图像、音频等）
│   ├── relay_adaptor.go # 渠道适配器工厂
│   └── channel/         # 各服务商的适配器（openai/、claude/、gemini/ 等）
├── model/               # GORM 模型 + 数据库操作（User、Channel、Token、Log 等）
├── middleware/          # Gin 中间件（认证、限流、分发器等）
├── service/             # 业务逻辑（配额、计费、Token 计数等）
├── dto/                 # 请求/响应数据传输对象
├── setting/             # 配置管理
│   ├── ratio_setting/   # 模型定价倍率
│   └── operation_setting/ # 运行时设置
├── constant/            # 常量（渠道类型、API 类型等）
├── common/              # 共享工具（Redis、邮件、日志等）
└── types/               # 共享类型定义
```

### 前端结构 (web/)

- React 18 + Vite + Bun
- UI 组件库：Semi Design (`@douyinfe/semi-ui`)
- 图表：VChart (`@visactor/react-vchart`)
- 国际化：i18next（支持中文、英文、法语、日语）
- 样式：Tailwind CSS

### 核心架构模式

**中继系统 (Relay System)**：网关的核心。发送到 `/v1/*` 的请求通过 `middleware.Distributor()` 选择渠道，然后由相应的适配器为每个服务商转换请求/响应。

**渠道适配器 (Channel Adaptors)**：每个 AI 服务商在 `relay/channel/` 中都有一个实现 `channel.Adaptor` 接口的适配器。`relay/relay_adaptor.go` 中的工厂根据 `constant.APIType*` 返回正确的适配器。

**中继中间件链**：
1. `TokenAuth()` - 验证 API 密钥
2. `Distribute()` - 根据模型、分组、优先级选择渠道
3. Handler（如 `Relay()`）- 通过适配器处理请求

**数据库**：通过 GORM 支持 SQLite（默认）、MySQL、PostgreSQL。启动时自动迁移模型。

**缓存**：Redis 用于分布式缓存；内存缓存作为备选。渠道缓存定期同步（`model.SyncChannelCache`）。

## 环境变量

关键变量（示例见 `docker-compose.yml`）：
- `SQL_DSN` - 数据库连接字符串
- `REDIS_CONN_STRING` - Redis 连接
- `SESSION_SECRET` - 多节点部署时必须设置
- `CRYPTO_SECRET` - 使用 Redis 时必须设置
- `PORT` - 服务端口（默认：3000）
- `GIN_MODE` - 开发时设置为 `debug`

## 支持的服务商

渠道类型定义在 `constant/channel.go`。适配器位于 `relay/channel/`：
- OpenAI、Azure、Claude/Anthropic、Gemini、AWS Bedrock
- 国内服务商：阿里（通义千问）、百度、腾讯、智谱、讯飞、Moonshot、DeepSeek
- 其他：Ollama、Cohere、Mistral、Perplexity、Cloudflare、SiliconFlow、xAI、Coze、Dify

## 添加新服务商

1. 在 `relay/channel/<provider>/adaptor.go` 中创建实现 `channel.Adaptor` 接口的适配器
2. 在 `constant/channel.go` 中添加渠道类型常量
3. 如需要，在 `constant/api_type.go` 中添加 API 类型常量
4. 在 `relay/relay_adaptor.go` 的 `GetAdaptor()` switch 中注册
5. 在 `common/endpoint_defaults.go` 中添加默认端点
