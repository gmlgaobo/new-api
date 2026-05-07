# New API - 项目技术文档

## 1. 仓库概览

New API 是一个下一代大语言模型 (LLM) 网关和 AI 资产管理系统，提供统一的 API 接口来聚合多个 AI 服务提供商的能力。

- **多模型统一接口**：支持 OpenAI、Claude、Gemini 等 40+ 主流 AI 服务提供商
- **灵活计费系统**：支持按使用量计费、多种支付方式集成
- **强大的路由能力**：智能重试、故障转移、负载均衡
- **多语言支持**：支持中文（简/繁）、英文、法文、日文
- **现代化 Web 界面**：提供直观的管理控制台
- **企业级特性**：用户权限管理、API Token、模型限流等

## 2. 目录结构

New API 采用清晰的分层架构，遵循 Go 语言项目的标准结构。项目分为后端 API 服务（Go）和两个前端 Web 界面（经典版和默认版）。核心逻辑集中在后端，包括路由、控制器、业务逻辑层和数据模型层。

```text
├── common/            # 通用工具和公共组件
├── constant/          # 常量定义
├── controller/        # HTTP 控制器层
├── dto/               # 数据传输对象
├── electron/          # Electron 桌面应用代码
├── i18n/              # 国际化资源
├── logger/            # 日志模块
├── middleware/        # Gin 中间件
├── model/             # 数据模型与数据库交互
├── oauth/             # OAuth 认证模块
├── pkg/               # 第三方库封装
├── relay/             # 代理转发核心模块
│   ├── channel/       # 各平台适配器
│   ├── common/        # 公共中继逻辑
│   └── helper/        # 辅助工具
├── router/            # 路由定义
├── service/           # 业务逻辑层
├── setting/           # 配置管理
├── types/             # 类型定义
├── web/               # 前端代码
│   ├── classic/       # 经典版界面
│   └── default/       # 默认版新界面
├── main.go            # 入口文件
└── go.mod             # Go 模块定义
```

* **controller/**：处理 HTTP 请求，进行参数验证、调用业务逻辑、返回响应
* **model/**：定义数据结构、数据库操作（如用户、渠道、Token）
* **relay/**：核心代理模块，包含所有 AI 服务提供商的适配代码
* **service/**：业务逻辑实现，包括计费、用户管理、渠道选择等
* **middleware/**：认证、限流、日志等中间件
* **web/**：两个前端界面，使用 React 构建

## 3. 系统架构与主流程

New API 采用经典的分层架构设计，从外部请求到最终响应的完整流程如下：

```
┌─────────────┐
│   客户端     │
└──────┬──────┘
       │ HTTP/WebSocket
       ▼
┌─────────────────────────┐
│     路由层 (router)     │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│   中间件层 (middleware)  │
│  - 认证                │
│  - 限流                │
│  - 日志                │
│  - 国际化              │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│   控制器层 (controller)  │
│  - 处理请求参数        │
│  - 调用业务逻辑        │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│    业务层 (service)      │
│  - 计费管理            │
│  - 渠道选择            │
│  - Token 管理         │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│    代理层 (relay)        │
│  - 请求格式转换        │
│  - 平台适配            │
│  - 响应处理            │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│ 第三方 AI 服务提供商    │
└─────────────────────────┘
```

### 主要数据流向：

1. **请求接收与验证**：
   - 客户端发起 API 请求，经过认证中间件验证身份与权限
   - 记录请求元数据，开始计费流程

2. **渠道选择**：
   - 根据模型名称和用户分组，从可用渠道中智能选择最佳渠道
   - 支持负载均衡、故障转移、重试机制

3. **请求转发**：
   - 适配层将请求格式转换为目标平台所需格式
   - 流式响应处理、错误重试

4. **响应处理与计费**：
   - 处理响应，转换为统一格式返回客户端
   - 计算 Token 使用量并完成计费结算

### 关键设计原则：

- **插件化架构**：各 AI 服务提供商的适配代码独立封装，易于扩展
- **分层隔离**：路由、控制、业务、数据访问各层职责清晰
- **高可用性**：完善的重试和故障转移机制

## 4. 核心功能模块

### 4.1 代理转发（Relay）

代理转发是 New API 的核心功能，负责将请求路由到正确的 AI 服务提供商并转换协议格式。

**主要组件：**
- 适配器模式实现：`relay/channel/*/adaptor.go` 各平台独立适配器
- 统一请求处理：`relay/common/` 包含共享逻辑
- 支持多种 API 格式：OpenAI、Claude、Gemini 等

**关键流程：**
1. 根据模型名称和渠道类型选择对应的适配器
2. 将统一请求格式转换为平台特定格式
3. 转发到上游服务并处理响应（含流式处理）
4. 转换响应回统一格式返回客户端

### 4.2 计费系统

灵活的计费系统支持多种计费策略和支付方式。

**主要功能：**
- 模型定价配置：`model/pricing.go` 和 `setting/billing_setting/`
- 计费策略：按 Token 计费、按次计费、分组倍率等
- 支付集成：支持 Stripe、EPay、Waffo 等
- 订阅管理：套餐订阅、配额自动重置

**核心文件：**
- `service/billing.go`：计费核心逻辑
- `service/tiered_settle.go`：阶梯计费
- `service/subscription_reset_task.go`：配额重置任务

### 4.3 渠道管理

渠道管理模块负责 AI 服务提供商的配置、状态管理和智能选择。

**主要功能：**
- 渠道配置：密钥、模型映射、优先级设置
- 健康检查：自动检测渠道可用性
- 智能选择：负载均衡、权重分配、故障转移
- 多密钥支持：单个渠道可配置多个密钥轮询使用

**核心文件：**
- `service/channel_select.go`：渠道选择逻辑
- `controller/channel.go`：渠道管理控制器
- `model/channel.go`：渠道数据模型

### 4.4 用户认证与授权

完善的认证系统支持多种登录方式和权限控制。

**认证方式：**
- 用户名密码
- OAuth 登录（GitHub、Discord、LinuxDo 等）
- API Token
- Passkey

**权限模型：**
- 普通用户、管理员、超级管理员三级权限
- Token 级权限：模型限制、IP 白名单、分组限制

**核心文件：**
- `middleware/auth.go`：认证中间件
- `model/user.go`、`model/token.go`：用户和 Token 模型
- `oauth/`：OAuth 认证实现

### 4.5 前端界面

两个前端界面提供不同的用户体验：

**经典版（web/classic/）：**
- 使用 React + Vite 构建
- 功能完整，久经考验

**默认版（web/default/）：**
- 现代化设计，使用 React + TypeScript
- 组件化架构，更好的用户体验
- 支持多语言、深色/浅色主题

## 5. 核心 API/类/函数

### 5.1 入口与初始化

#### `main()` - 主入口函数
**位置**：`main.go`
**功能**：应用程序入口，负责初始化资源、启动 HTTP 服务器
**主要流程**：
- 加载环境变量和配置
- 初始化数据库和 Redis
- 加载路由和中间件
- 启动定时任务（渠道检测、配额重置等）
- 启动 Gin HTTP 服务

#### `InitResources()` - 资源初始化
**位置**：`main.go`
**功能**：初始化各项系统资源
**参数**：无
**返回值**：`error` - 初始化错误
**调用关系**：被 `main()` 调用，负责初始化日志、配置、数据库等核心组件

### 5.2 路由与请求处理

#### `SetRouter()` - 路由设置
**位置**：`router/main.go`
**功能**：配置所有 HTTP 路由
**参数**：
- `*gin.Engine` - Gin 引擎实例
- `ThemeAssets` - 前端资源
**主要路由组**：
- API 路由
- 仪表板路由
- 中继（Relay）路由
- Web 界面路由

#### `Relay()` - 中继请求处理器
**位置**：`controller/relay.go`
**功能**：处理所有 AI API 请求的核心入口
**参数**：
- `*gin.Context` - Gin 上下文
- `types.RelayFormat` - 请求格式类型
**流程**：
1. 验证请求并提取 Token
2. 检查敏感词
3. 估算 Token 数量
4. 预扣费
5. 选择渠道并转发请求
6. 处理响应并完成计费

### 5.3 数据模型

#### `User` - 用户模型
**位置**：`model/user.go`
**主要字段**：
- `Id` - 用户 ID
- `Username` - 用户名
- `Password` - 密码（加密存储）
- `Quota` - 剩余配额
- `Role` - 用户角色
- `Status` - 用户状态

**关键方法**：
- `GetById(id int) (*User, error)` - 通过 ID 获取用户
- `UpdateQuota(id int, quota int)` - 更新用户配额

#### `Channel` - 渠道模型
**位置**：`model/channel.go`
**主要字段**：
- `Id` - 渠道 ID
- `Type` - 渠道类型（OpenAI、Claude 等）
- `Key` - API 密钥（加密存储）
- `Models` - 支持的模型列表
- `Status` - 渠道状态
- `Priority` - 优先级
- `Weight` - 权重

**关键方法**：
- `GetRandomSatisfiedChannel(...)` - 获取可用渠道
- `TestChannel(...)` - 测试渠道可用性

#### `Token` - API Token 模型
**位置**：`model/token.go`
**主要字段**：
- `Id` - Token ID
- `Key` - Token 密钥
- `UserId` - 所属用户 ID
- `Name` - Token 名称
- `RemainQuota` - 剩余配额
- `ModelLimits` - 模型限制
- `IpLimits` - IP 限制

**关键方法**：
- `ValidateUserToken(key string) (*Token, error)` - 验证 Token
- `GetTokenByKey(key string, checkStatus bool) (*Token, error)` - 通过 Key 获取 Token

### 5.4 中继适配器

#### `BaseAdaptor` - 适配器基类
**位置**：`relay/channel/adapter.go`
**主要接口**：
- `GetModelList()` - 获取模型列表
- `GetTokenConsume(...)` - 获取 Token 消耗
- `DoRequest(...)` - 执行请求
- `DoStreamRequest(...)` - 执行流式请求

各平台适配器（如 OpenAI、Claude、Gemini）都实现了这个接口。

### 5.5 业务服务

#### `CacheGetRandomSatisfiedChannel()` - 智能渠道选择
**位置**：`service/channel_select.go`
**功能**：根据模型和用户分组选择最佳可用渠道
**参数**：
- `*RetryParam` - 重试参数
**返回值**：
- `*model.Channel` - 选中的渠道
- `string` - 选中的分组
- `error` - 错误信息

**算法逻辑**：
1. 获取用户分组的所有可用渠道
2. 按优先级和权重筛选
3. 排除已失败的渠道（重试场景）
4. 随机选择符合条件的渠道

#### `PreConsumeBilling()` - 预扣费
**位置**：`service/pre_consume_quota.go`
**功能**：在请求前预先扣除配额
**参数**：
- `*gin.Context` - Gin 上下文
- `int` - 预计扣除配额
- `*relaycommon.RelayInfo` - 中继信息
**返回值**：
- `*types.NewAPIError` - 错误信息

#### `SettleBilling()` - 计费结算
**位置**：`service/billing.go`
**功能**：根据实际使用完成最终计费
**参数**：
- `*gin.Context` - Gin 上下文
- `*relaycommon.RelayInfo` - 中继信息
- `int` - 实际配额消耗

## 6. 技术栈与依赖

| 类别 | 技术/库 | 用途 |
|------|---------|------|
| **语言** | Go 1.21+ | 后端开发语言 |
| **Web 框架** | Gin | HTTP 服务器和路由 |
| **ORM** | GORM | 数据库操作 |
| **数据库** | SQLite/MySQL/PostgreSQL | 数据存储 |
| **缓存** | Redis | 缓存和会话存储 |
| **前端** | React + TypeScript + Vite | 默认版 UI |
| **前端** | React + JavaScript + Vite | 经典版 UI |
| **支付** | Stripe、EPay、Waffo | 支付集成 |
| **认证** | OAuth2、Passkey | 用户认证 |
| **国际化** | go-i18n | 多语言支持 |
| **监控** | Pyroscope | 性能分析 |
| **容器化** | Docker | 部署和分发 |

## 7. 关键模块与典型用例

### 7.1 API 调用代理

**功能说明**：最核心的用例，通过统一接口调用各类 AI 服务。

**配置与依赖**：
- 需要配置至少一个可用的 AI 服务渠道
- 用户需要有足够的配额

**使用示例**：

```bash
# 使用 OpenAI 兼容格式调用
curl https://api.newapi.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-xxx" \
  -d '{
    "model": "gpt-4",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

### 7.2 渠道管理

**功能说明**：管理员可以添加、配置、测试 AI 服务渠道。

**配置与依赖**：
- 管理员权限
- 各 AI 服务提供商的 API 密钥

**典型流程**：
1. 登录管理后台
2. 进入「渠道」管理页面
3. 点击「新建渠道」
4. 选择渠道类型，填入 API 密钥
5. 配置支持的模型和优先级
6. 测试渠道可用性

## 8. 配置、部署与开发

### 8.1 环境变量

主要配置项（完整列表见 `.env.example`）：

| 变量名 | 说明 | 默认值 |
|--------|------|--------|
| `SESSION_SECRET` | Session 密钥 | - |
| `SQL_DSN` | 数据库连接字符串 | `sqlite.db` |
| `REDIS_CONN_STRING` | Redis 连接字符串 | - |
| `PORT` | 服务端口 | `3000` |
| `STREAMING_TIMEOUT` | 流式请求超时（秒） | `300` |

### 8.2 Docker 部署

使用 Docker Compose 部署（推荐）：

```yaml
version: '3'

services:
  new-api:
    image: calciumion/new-api:latest
    container_name: new-api
    restart: always
    ports:
      - "3000:3000"
    environment:
      - SQL_DSN=root:password@tcp(mysql:3306)/newapi
      - TZ=Asia/Shanghai
    volumes:
      - ./data:/data
    depends_on:
      - mysql

  mysql:
    image: mysql:8.0
    container_name: new-api-mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: newapi
    volumes:
      - mysql-data:/var/lib/mysql

volumes:
  mysql-data:
```

### 8.3 开发环境设置

1. 克隆代码库
2. 安装 Go 1.21+ 和 Node.js 18+
3. 后端开发：
   ```bash
   go mod download
   go run main.go
   ```
4. 前端开发：
   ```bash
   cd web/default
   npm install
   npm run dev
   ```

## 9. 监控与维护

### 9.1 日志系统

New API 具有完善的日志系统：
- 应用日志：记录系统运行状态
- 请求日志：记录 API 调用详情
- 错误日志：记录错误信息（可配置保存到数据库）

### 9.2 性能监控

支持使用 Pyroscope 进行性能分析：
- 配置 `PYROSCOPE_URL` 启用
- 可监控 CPU、内存等指标

### 9.3 常见问题排查

| 问题 | 可能原因 | 排查方法 |
|------|----------|----------|
| 渠道不可用 | 密钥无效、网络问题、模型不匹配 | 检查渠道配置、测试渠道连接 |
| 计费异常 | 模型价格配置错误、倍率设置问题 | 检查模型价格和分组倍率 |
| 认证失败 | Token 无效、用户被禁用 | 检查 Token 状态和用户状态 |

## 10. 总结与亮点回顾

New API 是一个功能完善、架构清晰的 AI 服务网关系统，其核心亮点包括：

1. **高度可扩展的适配器架构**：通过统一的适配器接口，轻松支持新的 AI 服务提供商
2. **灵活的计费系统**：支持多种计费策略，满足不同商业场景需求
3. **高可用设计**：完善的重试、故障转移机制，确保服务稳定
4. **现代化技术栈**：使用最新的 Go、React 技术，保持代码质量和开发效率
5. **多语言支持**：完善的国际化架构，易于扩展新语言

New API 不仅是一个技术项目，更是连接用户与 AI 服务的桥梁，为 AI 应用的开发和部署提供了强大的基础设施支持。
