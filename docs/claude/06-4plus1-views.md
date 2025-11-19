# Chat Nio 4+1 架构视图

本文档采用 Philippe Kruchten 的 **4+1 架构视图模型**，从五个不同的视角全面描述 Chat Nio 系统架构。

---

## 4+1 视图模型概述

```mermaid
graph TB
    subgraph "4+1 Architecture Views"
        UV[用例视图<br/>Use Case View<br/>功能需求]
        LV[逻辑视图<br/>Logical View<br/>功能实现]
        PV[进程视图<br/>Process View<br/>并发与性能]
        IV[实现视图<br/>Implementation View<br/>代码组织]
        DV[部署视图<br/>Deployment View<br/>物理部署]
    end

    UV -.驱动.-> LV
    UV -.驱动.-> PV
    UV -.驱动.-> IV
    UV -.驱动.-> DV

    style UV fill:#FFE5B4,stroke:#333,stroke-width:3px
    style LV fill:#B4D7FF
    style PV fill:#B4FFD7
    style IV fill:#FFB4D7
    style DV fill:#E5B4FF
```

| 视图 | 关注点 | 主要用户 |
|------|--------|----------|
| **用例视图** | 系统功能、外部交互 | 终端用户、产品经理、测试人员 |
| **逻辑视图** | 功能分解、类和对象 | 架构师、开发人员 |
| **进程视图** | 并发、同步、性能 | 系统工程师、性能工程师 |
| **实现视图** | 代码组织、模块结构 | 开发人员、配置管理员 |
| **部署视图** | 物理部署、网络拓扑 | 运维工程师、系统管理员 |

---

## 1. 用例视图 (Use Case View)

### 1.1 系统参与者

```mermaid
graph LR
    subgraph 外部参与者
        User[普通用户]
        Premium[付费用户]
        Admin[管理员]
        APIClient[API客户端]
    end

    subgraph 外部系统
        OpenAI[OpenAI API]
        Claude[Claude API]
        Gemini[Gemini API]
        Other[其他AI提供商]
    end

    subgraph Chat Nio 系统
        System[Chat Nio<br/>核心系统]
    end

    User --> System
    Premium --> System
    Admin --> System
    APIClient --> System

    System --> OpenAI
    System --> Claude
    System --> Gemini
    System --> Other

    style System fill:#90EE90,stroke:#333,stroke-width:3px
```

### 1.2 核心用例图

```mermaid
graph TB
    subgraph "Chat Nio 系统"
        UC1[聊天对话]
        UC2[图像生成]
        UC3[视频生成]
        UC4[对话管理]
        UC5[用户认证]
        UC6[配额管理]
        UC7[订阅管理]
        UC8[渠道管理]
        UC9[用户管理]
        UC10[统计分析]
        UC11[Web搜索增强]
        UC12[API调用]
    end

    User((普通用户))
    Premium((付费用户))
    Admin((管理员))
    API((API客户端))

    User --> UC1
    User --> UC4
    User --> UC5
    User --> UC6

    Premium --> UC1
    Premium --> UC2
    Premium --> UC3
    Premium --> UC4
    Premium --> UC7
    Premium --> UC11

    Admin --> UC8
    Admin --> UC9
    Admin --> UC10
    Admin --> UC6

    API --> UC12
    API --> UC1

    UC1 -.include.-> UC5
    UC12 -.include.-> UC5
    UC1 -.extend.-> UC11
    UC6 -.include.-> UC5

    style User fill:#FFE5B4
    style Premium fill:#B4D7FF
    style Admin fill:#FFB4D7
    style API fill:#B4FFD7
```

### 1.3 详细用例说明

#### UC1: 聊天对话

**参与者**: 普通用户、付费用户、API客户端

**前置条件**:
- 用户已登录
- 配额充足

**主流程**:
1. 用户选择 AI 模型（gpt-4, claude-3 等）
2. 用户输入消息
3. 系统验证配额
4. 系统选择可用渠道
5. 系统调用 AI 提供商 API
6. 系统流式返回响应
7. 系统扣除配额
8. 系统保存对话历史

**扩展流程**:
- 3a. 配额不足 → 提示用户充值或订阅
- 5a. 渠道失败 → 自动切换下一个渠道
- 5b. 所有渠道失败 → 返回错误信息

**特殊需求**:
- 响应延迟 < 3秒（首字延迟）
- 支持流式输出（SSE）
- 支持 WebSocket 连接

---

#### UC2: 图像生成

**参与者**: 付费用户

**前置条件**:
- 用户已订阅或有足够配额
- 模型支持图像生成（DALL-E, Midjourney）

**主流程**:
1. 用户输入图像描述（prompt）
2. 用户选择尺寸和数量
3. 系统验证权限和配额
4. 系统调用图像生成 API
5. 系统轮询生成状态
6. 生成完成后返回图像 URL
7. 系统扣除配额

**扩展流程**:
- 5a. 生成超时 → 返回失败，不扣费
- 6a. 内容违规 → 返回错误提示

---

#### UC3: 视频生成

**参与者**: 付费用户

**前置条件**:
- 用户已订阅高级计划
- 模型支持视频生成（Sora）

**主流程**:
1. 用户输入视频描述
2. 用户选择视频参数（时长、分辨率）
3. 系统验证权限
4. 系统提交异步任务
5. 系统返回任务 ID
6. 用户轮询任务状态
7. 生成完成后返回视频 URL
8. 系统扣除配额

**特殊需求**:
- 任务保存时间 24 小时
- 支持任务取消

---

#### UC8: 渠道管理

**参与者**: 管理员

**前置条件**:
- 管理员已登录

**主流程**:
1. 管理员查看渠道列表
2. 管理员创建/编辑渠道
3. 配置渠道参数（模型、优先级、权重）
4. 配置 API 密钥和端点
5. 启用/禁用渠道
6. 系统重新加载渠道配置

**特殊需求**:
- 配置热更新（无需重启）
- 支持渠道性能监控

---

#### UC12: API 调用

**参与者**: API 客户端（开发者）

**前置条件**:
- 已获取 API Key

**主流程**:
1. 客户端携带 API Key 发起请求
2. 系统验证 API Key
3. 系统检查调用频率限制
4. 系统处理请求（同 UC1）
5. 系统返回 OpenAI 兼容格式响应
6. 系统记录 API 调用日志

**特殊需求**:
- 完全兼容 OpenAI API 格式
- 支持批量请求
- 频率限制：100 req/min

---

## 2. 逻辑视图 (Logical View)

### 2.1 系统分层架构

```mermaid
graph TB
    subgraph "表示层 Presentation Layer"
        UI[React 前端界面]
        API[RESTful API]
        WS[WebSocket API]
    end

    subgraph "应用层 Application Layer"
        AuthCtrl[认证控制器]
        ChatCtrl[聊天控制器]
        ImageCtrl[图像控制器]
        VideoCtrl[视频控制器]
        AdminCtrl[管理控制器]
    end

    subgraph "业务逻辑层 Business Logic Layer"
        AuthSvc[认证服务]
        QuotaSvc[配额服务]
        ChannelSvc[渠道服务]
        AdapterSvc[适配器服务]
        CacheSvc[缓存服务]
    end

    subgraph "数据访问层 Data Access Layer"
        UserRepo[用户仓储]
        ConvRepo[对话仓储]
        QuotaRepo[配额仓储]
        SubRepo[订阅仓储]
    end

    subgraph "基础设施层 Infrastructure Layer"
        MySQL[(MySQL)]
        Redis[(Redis)]
        OpenAI[OpenAI API]
        Claude[Claude API]
    end

    UI --> ChatCtrl
    API --> ChatCtrl
    WS --> ChatCtrl
    API --> AuthCtrl
    API --> AdminCtrl

    ChatCtrl --> AuthSvc
    ChatCtrl --> QuotaSvc
    ChatCtrl --> ChannelSvc
    ImageCtrl --> ChannelSvc
    VideoCtrl --> ChannelSvc

    ChannelSvc --> AdapterSvc
    ChannelSvc --> CacheSvc
    AuthSvc --> UserRepo
    QuotaSvc --> QuotaRepo

    UserRepo --> MySQL
    ConvRepo --> MySQL
    QuotaRepo --> MySQL
    SubRepo --> MySQL
    CacheSvc --> Redis

    AdapterSvc --> OpenAI
    AdapterSvc --> Claude

    style UI fill:#FFE5B4
    style ChatCtrl fill:#B4D7FF
    style ChannelSvc fill:#B4FFD7
    style MySQL fill:#FFB4D7
```

### 2.2 核心类图

```mermaid
classDiagram
    class User {
        +int Id
        +string Username
        +string Email
        +string Password
        +bool IsAdmin
        +bool IsBanned
        +string Token
        +UseQuota(amount)
        +GetQuota()
        +Subscribe(plan)
    }

    class Quota {
        +int UserId
        +float64 Total
        +float64 Used
        +float64 Available()
        +Deduct(amount)
        +Recharge(amount)
    }

    class Subscription {
        +int UserId
        +int Level
        +DateTime ExpiredAt
        +bool IsActive()
        +Renew(months)
    }

    class Channel {
        +int Id
        +string Type
        +string Name
        +[]string Models
        +int Priority
        +int Weight
        +bool State
        +int Retry
        +string Secret
        +IsHit(model)
        +IsHitGroup(group)
        +GetModelReflect(model)
    }

    class ChannelManager {
        +[]Channel Sequence
        +[]string Models
        +map PreflightSequence
        +Load()
        +GetTicker(model, group)
        +HitSequence(model)
    }

    class Ticker {
        +[]Channel Sequence
        +int Cursor
        +Next()
        +IsDone()
        +GetChannelByPriority(priority)
    }

    class Adapter {
        <<interface>>
        +CreateStreamChatRequest(props, hook)
        +CreateVideoRequest(props, hook)
    }

    class OpenAIAdapter {
        +string Endpoint
        +string ApiKey
        +CreateStreamChatRequest(props, hook)
    }

    class ClaudeAdapter {
        +string Endpoint
        +string ApiKey
        +CreateStreamChatRequest(props, hook)
    }

    class ChatProps {
        +string Model
        +[]Message Messages
        +int MaxTokens
        +float Temperature
        +*int MaxRetries
        +int Current
    }

    class Buffer {
        +string Model
        +[]Message Messages
        +string Content
        +*ToolCalls ToolCalls
        +int InputTokens
        +int OutputTokens
        +WriteChunk(chunk)
        +GetQuota()
        +CountTokens()
    }

    class Message {
        +string Role
        +string Content
        +*ToolCalls ToolCalls
        +*FunctionCall FunctionCall
    }

    User "1" --> "1" Quota : has
    User "1" --> "0..1" Subscription : has
    ChannelManager "1" --> "*" Channel : manages
    ChannelManager --> Ticker : creates
    Ticker --> Channel : iterates
    Adapter <|.. OpenAIAdapter : implements
    Adapter <|.. ClaudeAdapter : implements
    Adapter --> ChatProps : uses
    Buffer --> Message : contains
```

### 2.3 聊天请求序列图

```mermaid
sequenceDiagram
    actor User as 用户
    participant Ctrl as ChatController
    participant Auth as AuthService
    participant Quota as QuotaService
    participant Channel as ChannelService
    participant Adapter as AdapterService
    participant AI as AI Provider

    User->>Ctrl: POST /v1/chat/completions
    activate Ctrl

    Ctrl->>Auth: VerifyToken(token)
    activate Auth
    Auth-->>Ctrl: User
    deactivate Auth

    Ctrl->>Quota: CheckQuota(user, model)
    activate Quota
    Quota-->>Ctrl: OK / Error
    deactivate Quota

    Ctrl->>Channel: NewChatRequest(props)
    activate Channel

    Channel->>Channel: GetTicker(model, group)
    Channel->>Channel: Ticker.Next()

    loop 遍历渠道直到成功
        Channel->>Adapter: NewChatRequest(channel, props)
        activate Adapter

        Adapter->>AI: HTTP POST /v1/chat/completions
        activate AI

        loop 流式响应
            AI-->>Adapter: SSE: data chunk
            Adapter-->>Channel: chunk
            Channel-->>Ctrl: chunk
            Ctrl-->>User: SSE: chunk
        end

        AI-->>Adapter: [DONE]
        deactivate AI

        alt 成功
            Adapter-->>Channel: Success
        else 失败
            Adapter-->>Channel: Error
            Channel->>Channel: Ticker.Next()
        end

        deactivate Adapter
    end

    Channel-->>Ctrl: Response
    deactivate Channel

    Ctrl->>Quota: DeductQuota(user, amount)
    activate Quota
    Quota-->>Ctrl: OK
    deactivate Quota

    Ctrl-->>User: [DONE]
    deactivate Ctrl
```

### 2.4 状态图：渠道选择

```mermaid
stateDiagram-v2
    [*] --> 初始化
    初始化 --> 获取Ticker: GetTicker(model, group)
    获取Ticker --> 检查是否有渠道: ticker.IsEmpty()

    检查是否有渠道 --> 无可用渠道: 空
    检查是否有渠道 --> 选择渠道: 非空

    选择渠道 --> 按优先级选择: ticker.Next()
    按优先级选择 --> 同优先级加权随机: 多个同优先级
    按优先级选择 --> 返回唯一渠道: 单个渠道

    同优先级加权随机 --> 发起请求
    返回唯一渠道 --> 发起请求

    发起请求 --> 请求成功: adapter.NewChatRequest()
    发起请求 --> 请求失败

    请求失败 --> 检查重试: 判断错误类型
    检查重试 --> QPS限流重试: QPS限流
    检查重试 --> 网络重试: 网络错误
    检查重试 --> 切换渠道: 其他错误

    QPS限流重试 --> 等待500ms: sleep
    等待500ms --> 发起请求: 递归重试

    网络重试 --> 检查重试次数: Current < MaxRetries
    检查重试次数 --> 发起请求: 是
    检查重试次数 --> 切换渠道: 否

    切换渠道 --> 检查是否遍历完: ticker.IsDone()
    检查是否遍历完 --> 选择渠道: 否
    检查是否遍历完 --> 渠道耗尽: 是

    请求成功 --> [*]
    无可用渠道 --> [*]
    渠道耗尽 --> [*]
```

---

## 3. 进程视图 (Process View)

### 3.1 系统并发架构

```mermaid
graph TB
    subgraph "Gin HTTP Server"
        G1[Goroutine 1<br/>请求处理线程]
        G2[Goroutine 2<br/>请求处理线程]
        G3[Goroutine N<br/>请求处理线程]
    end

    subgraph "流式响应处理"
        S1[Goroutine<br/>AI API 请求]
        S2[Goroutine<br/>AI API 请求]
        C1[Channel<br/>消息队列]
        C2[Channel<br/>消息队列]
    end

    subgraph "数据库连接池"
        DB1[MySQL Conn 1]
        DB2[MySQL Conn 2]
        DB3[MySQL Conn N]
        Pool[Connection Pool<br/>Max: 512<br/>Idle: 64]
    end

    subgraph "Redis 连接池"
        R1[Redis Conn 1]
        R2[Redis Conn 2]
        R3[Redis Conn N]
        RPool[Redis Pool]
    end

    subgraph "后台任务"
        BG1[数据库连接监控]
        BG2[统计数据收集]
        BG3[缓存清理]
    end

    G1 --> S1
    G2 --> S2
    S1 --> C1
    S2 --> C2
    C1 --> G1
    C2 --> G2

    G1 --> Pool
    G2 --> Pool
    G3 --> Pool
    Pool --> DB1
    Pool --> DB2
    Pool --> DB3

    G1 --> RPool
    G2 --> RPool
    RPool --> R1
    RPool --> R2
    RPool --> R3

    style G1 fill:#FFE5B4
    style S1 fill:#B4D7FF
    style C1 fill:#B4FFD7
    style Pool fill:#FFB4D7
```

### 3.2 流式响应并发模型

```mermaid
sequenceDiagram
    participant Main as 主Goroutine
    participant Worker as 工作Goroutine
    participant Chan as Channel队列
    participant AI as AI Provider

    Main->>Worker: go func() 启动
    activate Worker

    Main->>Main: c.Stream() 开始SSE
    activate Main

    Worker->>AI: HTTP POST (流式请求)
    activate AI

    loop 流式响应循环
        AI-->>Worker: SSE: data chunk
        Worker->>Worker: 解析 chunk
        Worker->>Chan: partial <- chunk
        Chan-->>Main: <- partial
        Main->>Main: c.Render(SSE event)
        Main-->>Client: SSE: chunk
    end

    AI-->>Worker: [DONE]
    deactivate AI

    Worker->>Chan: close(partial)
    deactivate Worker

    Main->>Main: 检测 channel 关闭
    Main->>Main: c.Render([DONE])
    Main-->>Client: SSE: [DONE]
    deactivate Main
```

### 3.3 进程间通信

```mermaid
graph LR
    subgraph "Chat Nio 进程"
        M[Main Thread]
        G1[Goroutine Pool]
        G2[Database Worker]
        G3[Cache Worker]
    end

    subgraph "外部进程"
        MySQL[(MySQL<br/>3306)]
        Redis[(Redis<br/>6379)]
        OpenAI[OpenAI API<br/>HTTPS]
    end

    M --> G1
    G1 --> G2
    G1 --> G3

    G2 -.TCP.-> MySQL
    G3 -.TCP.-> Redis
    G1 -.HTTPS.-> OpenAI

    style M fill:#FFE5B4
    style G1 fill:#B4D7FF
    style MySQL fill:#FFB4D7
    style Redis fill:#B4FFD7
```

### 3.4 性能指标

| 指标 | 目标值 | 实现方式 |
|------|--------|----------|
| **并发请求数** | 1000+ QPS | Gin + Goroutine 池 |
| **首字延迟** | < 3秒 | 流式响应 + 多渠道冗余 |
| **平均响应时间** | < 5秒（完整响应） | 缓存 + 负载均衡 |
| **数据库连接** | 512 并发连接 | 连接池管理 |
| **缓存命中率** | > 30% | Redis 缓存 + MD5 哈希 |
| **错误率** | < 1% | 多渠道自动重试 |

### 3.5 线程同步机制

```go
// Goroutine + Channel 同步
func sendStreamResponse(c *gin.Context, props *ChatProps) {
    partial := make(chan Response)  // 无缓冲 channel

    // 工作 Goroutine
    go func() {
        err := channel.NewChatRequest(props, func(chunk *Chunk) error {
            partial <- Response{Chunk: chunk}  // 阻塞发送
            return nil
        })
        close(partial)  // 关闭 channel
    }()

    // 主 Goroutine (SSE 输出)
    c.Stream(func(w io.Writer) bool {
        if resp, ok := <-partial; ok {  // 阻塞接收
            c.Render(-1, NewEvent(resp))
            return true  // 继续流式传输
        }
        return false  // 结束
    })
}
```

---

## 4. 实现视图 (Implementation View)

### 4.1 包结构图

```mermaid
graph TB
    subgraph "main"
        Main[main.go<br/>程序入口]
    end

    subgraph "adapter 适配器层"
        Adapter[adapter.go<br/>工厂注册]
        Request[request.go<br/>请求分发]

        subgraph "adapter/openai"
            OpenAI[chat.go<br/>OpenAI实现]
        end

        subgraph "adapter/claude"
            Claude[chat.go<br/>Claude实现]
        end

        subgraph "adapter/common"
            Common[types.go<br/>公共接口]
        end
    end

    subgraph "channel 渠道层"
        Manager[manager.go<br/>渠道管理器]
        Ticker[ticker.go<br/>迭代器]
        Worker[worker.go<br/>请求分发]
        Charge[charge.go<br/>计费管理]
    end

    subgraph "manager 业务层"
        ChatCompl[chat_completions.go<br/>聊天API]
        Images[images.go<br/>图像生成]
        Videos[videos.go<br/>视频生成]

        subgraph "manager/conversation"
            Conv[conversation.go<br/>对话同步]
        end
    end

    subgraph "auth 认证层"
        Auth[auth.go<br/>JWT认证]
        Quota[quota.go<br/>配额管理]
        Sub[subscription.go<br/>订阅管理]
    end

    subgraph "connection 数据层"
        DB[database.go<br/>数据库连接]
        Cache[cache.go<br/>Redis缓存]
        Migration[db_migration.go<br/>迁移]
    end

    subgraph "utils 工具层"
        Config[config.go<br/>配置加载]
        Buffer[buffer.go<br/>响应缓冲]
        Token[tokenizer.go<br/>Token计数]
        WS[websocket.go<br/>WebSocket]
    end

    subgraph "globals 全局定义"
        Types[types.go<br/>类型定义]
        Channel[channel.go<br/>渠道常量]
    end

    Main --> Manager
    Main --> Auth
    Main --> Adapter

    Manager --> Worker
    ChatCompl --> Worker
    Images --> Worker
    Videos --> Worker

    Worker --> Manager
    Worker --> Request

    Request --> Adapter
    Request --> OpenAI
    Request --> Claude

    OpenAI --> Common
    Claude --> Common

    Auth --> DB
    Manager --> Auth
    Manager --> Buffer

    DB --> Cache
    Manager --> Config

    style Main fill:#FFE5B4
    style Manager fill:#B4D7FF
    style Worker fill:#B4FFD7
    style DB fill:#FFB4D7
```

### 4.2 模块依赖矩阵

| 模块 ↓ 依赖 → | main | manager | channel | adapter | auth | connection | utils | globals |
|---------------|------|---------|---------|---------|------|------------|-------|---------|
| **main** | - | ✓ | ✓ | ✓ | ✓ | - | ✓ | - |
| **manager** | - | - | ✓ | - | ✓ | - | ✓ | ✓ |
| **channel** | - | - | - | ✓ | - | - | ✓ | ✓ |
| **adapter** | - | - | - | - | - | - | ✓ | ✓ |
| **auth** | - | - | - | - | - | ✓ | ✓ | ✓ |
| **connection** | - | - | - | - | - | - | ✓ | ✓ |
| **utils** | - | - | - | - | - | - | - | ✓ |
| **globals** | - | - | - | - | - | - | - | - |

**依赖规则**:
- ✓ 表示直接依赖
- 依赖方向：行依赖列
- globals 和 utils 为基础模块，被所有模块依赖
- 无循环依赖

### 4.3 构建与部署流程

```mermaid
flowchart TD
    A[源代码] --> B{选择构建方式}

    B -->|本地构建| C[go build]
    B -->|Docker构建| D[docker build]

    C --> E[编译 Go 代码]
    E --> F[链接依赖库]
    F --> G[生成二进制文件<br/>chatnio]

    D --> H[多阶段构建]
    H --> I[Stage 1: 前端构建<br/>pnpm build]
    I --> J[Stage 2: Go 构建<br/>go build]
    J --> K[Stage 3: 最小运行镜像<br/>alpine:latest]

    G --> L[本地运行<br/>./chatnio]
    K --> M[Docker运行<br/>docker run]

    L --> N[服务启动<br/>:8094]
    M --> N

    subgraph "前端构建"
        P[npm install] --> Q[vite build]
        Q --> R[dist/]
    end

    I --> P

    style B fill:#FFE5B4
    style G fill:#B4D7FF
    style K fill:#B4FFD7
```

### 4.4 代码组织规范

**文件命名规范**:
```
包名/
├── package.go          # 包的主要逻辑
├── types.go            # 类型定义
├── controller.go       # HTTP 控制器
├── router.go           # 路由注册
├── instance.go         # 实例初始化
└── package_test.go     # 单元测试
```

**import 顺序**:
```go
import (
    // 1. 标准库
    "fmt"
    "time"

    // 2. 第三方库
    "github.com/gin-gonic/gin"
    "github.com/spf13/viper"

    // 3. 本项目包
    "chat/adapter"
    "chat/globals"
    "chat/utils"
)
```

---

## 5. 部署视图 (Deployment View)

### 5.1 物理架构图

```mermaid
graph TB
    subgraph "客户端层"
        Browser[Web浏览器]
        Mobile[移动应用<br/>PWA]
        CLI[CLI工具]
    end

    subgraph "负载均衡层"
        LB[Nginx<br/>负载均衡器<br/>:80/:443]
    end

    subgraph "应用服务器集群"
        subgraph "服务器 1"
            App1[Chat Nio<br/>:8094]
        end

        subgraph "服务器 2"
            App2[Chat Nio<br/>:8094]
        end

        subgraph "服务器 N"
            AppN[Chat Nio<br/>:8094]
        end
    end

    subgraph "数据库层"
        subgraph "MySQL 主从集群"
            MySQL_M[(MySQL Master<br/>:3306)]
            MySQL_S1[(MySQL Slave 1<br/>:3306)]
            MySQL_S2[(MySQL Slave 2<br/>:3306)]
        end

        subgraph "Redis 集群"
            Redis_M[(Redis Master<br/>:6379)]
            Redis_S1[(Redis Slave 1<br/>:6379)]
            Redis_S2[(Redis Slave 2<br/>:6379)]
        end
    end

    subgraph "外部服务"
        OpenAI[OpenAI API<br/>api.openai.com]
        Claude[Claude API<br/>api.anthropic.com]
        Gemini[Gemini API<br/>generativelanguage.googleapis.com]
    end

    Browser -.HTTPS.-> LB
    Mobile -.HTTPS.-> LB
    CLI -.HTTPS.-> LB

    LB --> App1
    LB --> App2
    LB --> AppN

    App1 --> MySQL_M
    App2 --> MySQL_M
    AppN --> MySQL_M

    MySQL_M -.复制.-> MySQL_S1
    MySQL_M -.复制.-> MySQL_S2

    App1 --> Redis_M
    App2 --> Redis_M
    AppN --> Redis_M

    Redis_M -.复制.-> Redis_S1
    Redis_M -.复制.-> Redis_S2

    App1 -.HTTPS.-> OpenAI
    App1 -.HTTPS.-> Claude
    App1 -.HTTPS.-> Gemini

    App2 -.HTTPS.-> OpenAI
    App2 -.HTTPS.-> Claude
    App2 -.HTTPS.-> Gemini

    style Browser fill:#FFE5B4
    style LB fill:#B4D7FF
    style App1 fill:#B4FFD7
    style MySQL_M fill:#FFB4D7
```

### 5.2 Docker Compose 部署

```yaml
version: '3.8'

services:
  # Nginx 反向代理
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./certs:/etc/nginx/certs
    depends_on:
      - app
    networks:
      - frontend

  # Chat Nio 应用 (可水平扩展)
  app:
    image: chatnio/chatnio:latest
    deploy:
      replicas: 3  # 3个实例
      resources:
        limits:
          cpus: '2'
          memory: 2G
    environment:
      - MYSQL_HOST=mysql
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
      - REDIS_HOST=redis
      - SECRET=${JWT_SECRET}
    volumes:
      - ./config:/config
      - ./logs:/logs
    depends_on:
      - mysql
      - redis
    networks:
      - frontend
      - backend

  # MySQL 数据库
  mysql:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_PASSWORD}
      - MYSQL_DATABASE=chatnio
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - backend
    command: --default-authentication-plugin=mysql_native_password

  # Redis 缓存
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - backend

volumes:
  mysql_data:
  redis_data:

networks:
  frontend:
  backend:
```

### 5.3 Kubernetes 部署架构

```mermaid
graph TB
    subgraph "Kubernetes 集群"
        subgraph "Ingress"
            Ingress[Ingress Controller<br/>Nginx/Traefik]
        end

        subgraph "应用 Namespace"
            subgraph "Deployment: chatnio"
                Pod1[Pod 1<br/>chatnio:latest]
                Pod2[Pod 2<br/>chatnio:latest]
                Pod3[Pod 3<br/>chatnio:latest]
            end

            Service[Service: chatnio<br/>ClusterIP]
            HPA[HPA<br/>自动扩缩容<br/>2-10 Pods]
        end

        subgraph "数据 Namespace"
            subgraph "StatefulSet: mysql"
                MySQL[MySQL Pod<br/>PVC: 100Gi]
            end

            subgraph "StatefulSet: redis"
                Redis[Redis Pod<br/>PVC: 20Gi]
            end
        end

        subgraph "配置与密钥"
            ConfigMap[ConfigMap<br/>config.yaml]
            Secret[Secret<br/>API Keys, JWT]
        end
    end

    Ingress --> Service
    Service --> Pod1
    Service --> Pod2
    Service --> Pod3

    HPA -.监控.-> Pod1
    HPA -.扩缩.-> Service

    Pod1 --> MySQL
    Pod2 --> MySQL
    Pod3 --> MySQL

    Pod1 --> Redis
    Pod2 --> Redis
    Pod3 --> Redis

    Pod1 -.挂载.-> ConfigMap
    Pod1 -.挂载.-> Secret

    style Ingress fill:#FFE5B4
    style Service fill:#B4D7FF
    style Pod1 fill:#B4FFD7
    style MySQL fill:#FFB4D7
```

### 5.4 网络拓扑

```mermaid
graph LR
    subgraph "公网 Internet"
        User[用户]
    end

    subgraph "DMZ 区"
        FW1[防火墙<br/>WAF]
        LB[负载均衡器<br/>80/443]
    end

    subgraph "应用区 VLAN 10"
        App1[App Server 1<br/>192.168.10.11:8094]
        App2[App Server 2<br/>192.168.10.12:8094]
    end

    subgraph "数据区 VLAN 20"
        DB[MySQL<br/>192.168.20.10:3306]
        Cache[Redis<br/>192.168.20.11:6379]
    end

    subgraph "外部服务"
        AI[AI Provider APIs<br/>HTTPS]
    end

    User -->|HTTPS| FW1
    FW1 -->|HTTPS| LB
    LB -->|HTTP| App1
    LB -->|HTTP| App2

    App1 -->|TCP 3306| DB
    App2 -->|TCP 3306| DB
    App1 -->|TCP 6379| Cache
    App2 -->|TCP 6379| Cache

    App1 -->|HTTPS| AI
    App2 -->|HTTPS| AI

    style User fill:#FFE5B4
    style LB fill:#B4D7FF
    style App1 fill:#B4FFD7
    style DB fill:#FFB4D7
```

### 5.5 部署配置

#### 服务器规格推荐

| 角色 | CPU | 内存 | 磁盘 | 数量 |
|------|-----|------|------|------|
| **应用服务器** | 4 核 | 8 GB | 100 GB SSD | 2-5 |
| **MySQL** | 8 核 | 16 GB | 500 GB SSD | 1 主 + 1 从 |
| **Redis** | 4 核 | 8 GB | 50 GB SSD | 1 主 + 1 从 |
| **负载均衡器** | 2 核 | 4 GB | 20 GB | 1-2 |

#### 端口映射

```
外部端口          内部端口          服务
80            →   80            →   Nginx (HTTP)
443           →   443           →   Nginx (HTTPS)
              →   8094          →   Chat Nio (App)
              →   3306          →   MySQL
              →   6379          →   Redis
```

#### 防火墙规则

```bash
# 允许 HTTP/HTTPS
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# 应用服务器访问数据库
iptables -A INPUT -s 192.168.10.0/24 -p tcp --dport 3306 -j ACCEPT
iptables -A INPUT -s 192.168.10.0/24 -p tcp --dport 6379 -j ACCEPT

# 拒绝其他访问
iptables -A INPUT -p tcp --dport 3306 -j DROP
iptables -A INPUT -p tcp --dport 6379 -j DROP
```

---

## 6. 场景分析

### 6.1 场景 1: 高并发聊天请求

**场景描述**: 1000 个用户同时发起聊天请求

**各视图分析**:

**用例视图**:
- 参与者: 1000 个普通用户
- 用例: UC1 聊天对话

**逻辑视图**:
- ChatController 接收 1000 个请求
- ChannelService 分配到不同渠道
- 多个 Adapter 并发调用 AI API

**进程视图**:
- Gin 创建 1000 个 Goroutine 处理请求
- 每个请求启动独立的工作 Goroutine
- 数据库连接池复用 512 个连接

**实现视图**:
- manager/chat_completions.go 处理请求
- channel/worker.go 分发到渠道
- adapter/openai/chat.go 调用 API

**部署视图**:
- 负载均衡器分配到 3 个应用实例
- 每个实例处理约 333 个请求
- MySQL 承载 1000 次配额查询/更新

**性能指标**:
- 首字延迟: 2-3 秒
- 完整响应: 10-15 秒
- CPU 使用率: 60-80%
- 内存使用: 4-6 GB

---

### 6.2 场景 2: 渠道故障自动切换

**场景描述**: OpenAI 渠道故障，系统自动切换到 Claude

**各视图分析**:

**用例视图**:
- 用例: UC1 聊天对话
- 扩展流程: 5a. 渠道失败 → 自动切换

**逻辑视图**:
- Ticker.Next() 选择 OpenAI (优先级 1)
- OpenAI Adapter 返回错误
- Ticker.Next() 切换到 Claude (优先级 2)

**进程视图**:
- 主 Goroutine 检测到错误
- 不创建新线程，同步切换渠道
- 重用现有 HTTP 连接池

**实现视图**:
- channel/worker.go:21-30 (循环遍历渠道)
- adapter/request.go:27-48 (重试逻辑)

**部署视图**:
- 应用服务器检测到 OpenAI API 超时
- 切换到 Claude API (不同的网络连接)
- 用户无感知切换

**时间指标**:
- 故障检测: 5 秒（HTTP 超时）
- 切换时间: < 1 秒
- 总延迟增加: 5-6 秒

---

### 6.3 场景 3: 缓存命中

**场景描述**: 相同的请求命中 Redis 缓存

**各视图分析**:

**用例视图**:
- 用例: UC1 聊天对话
- 前置条件: 缓存中存在相同请求

**逻辑视图**:
- CacheService.PreflightCache() 查询 Redis
- 命中后直接返回，跳过 ChannelService

**进程视图**:
- 主 Goroutine 直接返回缓存数据
- 不启动工作 Goroutine
- Redis 连接池复用

**实现视图**:
- channel/worker.go:41-73 (PreflightCache)
- 缓存键: `chat-cache:{index}:{md5_hash}`

**部署视图**:
- 应用服务器查询 Redis 集群
- Redis 返回序列化的 Buffer 对象
- 不访问 AI Provider API

**性能指标**:
- 响应时间: < 100ms（vs 5-10s）
- 成本节省: 100%（无 API 调用）
- 缓存命中率: 30-40%

---

## 7. 关键决策记录

### 7.1 架构决策

| 决策 | 理由 | 权衡 |
|------|------|------|
| **Go + Gin 框架** | 高性能、并发支持好 | 学习曲线略陡 |
| **适配器模式** | 统一多提供商接口 | 增加抽象层 |
| **Ticker 迭代器** | 灵活的渠道选择策略 | 实现复杂度增加 |
| **Goroutine + Channel** | 原生并发支持 | 需要管理 Goroutine 生命周期 |
| **Redis 缓存** | 减少重复调用，降低成本 | 增加缓存一致性管理 |
| **MySQL 主数据库** | 成熟稳定、ACID 保证 | 需要维护数据库集群 |
| **前后端分离** | 独立开发、独立部署 | 跨域和认证复杂度增加 |

### 7.2 技术选型对比

**Web 框架对比**:
| 框架 | 性能 | 生态 | 学习曲线 | 选择理由 |
|------|------|------|----------|----------|
| **Gin** ✓ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | 高性能、简洁 |
| Echo | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | 功能类似 |
| Fiber | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ | 生态较弱 |

**数据库对比**:
| 数据库 | 性能 | 功能 | 运维成本 | 选择理由 |
|--------|------|------|----------|----------|
| **MySQL** ✓ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | 成熟稳定 |
| PostgreSQL | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | 功能更强 |
| MongoDB | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | 不适合事务 |

---

## 8. 总结

### 8.1 架构优势

1. **分层清晰**: 5 层架构（表示层、应用层、业务层、数据层、基础设施层）
2. **高可扩展**: 适配器模式支持快速接入新的 AI 提供商
3. **高可用**: 多渠道冗余、自动故障转移、缓存降级
4. **高性能**: Goroutine 并发、连接池复用、Redis 缓存
5. **灵活计费**: 支持多种计费模式（Token、次数、订阅）

### 8.2 质量属性

| 属性 | 策略 | 实现 |
|------|------|------|
| **性能** | 并发、缓存、连接池 | Goroutine + Redis + 连接池 |
| **可靠性** | 冗余、重试、降级 | 多渠道 + 自动重试 + 缓存 |
| **可扩展性** | 分层、接口、水平扩展 | 适配器模式 + 无状态应用 |
| **安全性** | 认证、加密、防注入 | JWT + HTTPS + 参数化查询 |
| **可维护性** | 模块化、文档、日志 | 包结构 + 详细文档 + 日志系统 |

### 8.3 未来演进方向

1. **微服务化**: 拆分为独立的认证服务、聊天服务、计费服务
2. **服务网格**: 引入 Istio/Linkerd 管理服务间通信
3. **消息队列**: 使用 Kafka/RabbitMQ 处理异步任务
4. **分布式追踪**: 集成 Jaeger/Zipkin 进行链路追踪
5. **智能调度**: 基于 AI 的渠道选择和负载预测

---

**文档版本**: v1.0
**创建日期**: 2024-01-15
**作者**: Claude
**审阅**: -
