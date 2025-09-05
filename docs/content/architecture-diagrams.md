# NetCorePal 框架最终一致性架构图

## 整体架构图

```mermaid
graph TB
    %% 客户端和API层
    Client[客户端] --> API[Web API]
    
    %% 命令处理层
    API --> CH[命令处理器<br/>CommandHandler]
    
    %% 本地事务边界
    subgraph "本地事务边界"
        CH --> UOW[工作单元<br/>UnitOfWork]
        UOW --> DB[(数据库)]
        
        %% 领域事件处理
        CH --> DE[领域事件<br/>DomainEvent]
        DE --> DEH[领域事件处理器<br/>DomainEventHandler]
        
        %% 集成事件发布
        DEH --> IEP[集成事件发布器<br/>CapIntegrationEventPublisher]
        IEP --> CAP[CAP组件]
        CAP --> OUTBOX[(Outbox表)]
        OUTBOX --> DB
    end
    
    %% 消息队列和远程服务
    CAP -.异步发布.-> MQ[消息队列<br/>RabbitMQ]
    MQ --> RS1[远程服务1]
    MQ --> RS2[远程服务2]
    MQ --> RS3[远程服务N]
    
    %% 集成事件处理
    RS1 --> IEH1[集成事件处理器1]
    RS2 --> IEH2[集成事件处理器2]
    RS3 --> IEH3[集成事件处理器N]
    
    %% 样式
    classDef localTx fill:#e1f5fe
    classDef async fill:#fff3e0
    classDef remote fill:#f3e5f5
    
    class UOW,DB,DE,DEH,IEP,CAP,OUTBOX localTx
    class MQ async
    class RS1,RS2,RS3,IEH1,IEH2,IEH3 remote
```

## 事件处理时序图

```mermaid
sequenceDiagram
    participant C as 客户端
    participant API as Web API
    participant CH as 命令处理器
    participant UOW as 工作单元
    participant DB as 数据库
    participant DEH as 领域事件处理器
    participant IEP as 集成事件发布器
    participant CAP as CAP组件
    participant MQ as 消息队列
    participant RS as 远程服务
    
    C->>API: 发送命令请求
    API->>CH: 处理命令
    
    Note over CH,CAP: 本地事务开始
    CH->>UOW: 开始事务
    UOW->>DB: 开始数据库事务
    
    CH->>CH: 执行业务逻辑
    CH-->>DEH: 触发领域事件
    
    DEH->>IEP: 发布集成事件
    IEP->>CAP: 存储事件到Outbox
    CAP->>DB: 持久化事件（同一事务）
    
    UOW->>DB: 提交事务
    Note over CH,CAP: 本地事务结束
    
    API-->>C: 返回响应
    
    Note over CAP,RS: 异步事件处理
    CAP-->>MQ: 异步发布事件
    MQ-->>RS: 投递集成事件
    RS->>RS: 处理集成事件
    
    alt 处理成功
        RS-->>MQ: 确认消息
    else 处理失败
        RS-->>MQ: 拒绝消息
        MQ-->>RS: 重试投递（最多10次）
    end
```

## 核心组件关系图

```mermaid
classDiagram
    %% 集成事件相关接口
    class IIntegrationEvent {
        <<interface>>
    }
    
    class IIntegrationEventPublisher {
        <<interface>>
        +PublishAsync(event, cancellationToken)
    }
    
    class IIntegrationEventHandler~T~ {
        <<interface>>
        +Handle(eventData, cancellationToken)
    }
    
    %% CAP实现
    class CapIntegrationEventPublisher {
        -ICapPublisher _capPublisher
        +PublishAsync(context)
    }
    
    %% 事务处理
    class IPublisherTransactionHandler {
        <<interface>>
        +BeginTransaction(context)
    }
    
    class CapSqlServerPublisherTransactionHandler {
        -Lazy~ICapPublisher~ _capBus
        +BeginTransaction(context)
    }
    
    %% 数据库上下文
    class AppDbContextBase {
        -IMediator _mediator
        -IPublisherTransactionHandler _publisherTransactionFactory
        +BeginTransaction()
        +SaveEntitiesAsync()
        +CommitAsync()
        +RollbackAsync()
    }
    
    %% 工作单元行为
    class CommandUnitOfWorkBehavior~TCommand,TResponse~ {
        -ITransactionUnitOfWork _unitOfWork
        +Handle(request, next, cancellationToken)
    }
    
    %% 关系
    IIntegrationEventPublisher <|-- CapIntegrationEventPublisher
    IPublisherTransactionHandler <|-- CapSqlServerPublisherTransactionHandler
    
    CapIntegrationEventPublisher --> ICapPublisher : uses
    AppDbContextBase --> IPublisherTransactionHandler : uses
    CommandUnitOfWorkBehavior --> ITransactionUnitOfWork : uses
    
    AppDbContextBase ..> CapSqlServerPublisherTransactionHandler : creates transaction
    CapIntegrationEventPublisher ..> IIntegrationEvent : publishes
    IIntegrationEventHandler ..> IIntegrationEvent : handles
```

## 数据流图

```mermaid
flowchart LR
    %% 业务数据流
    subgraph "业务数据流"
        BD[业务数据] --> TX[数据库事务]
        TX --> DB[(业务数据库)]
    end
    
    %% 事件数据流
    subgraph "事件数据流"
        DE[领域事件] --> IE[集成事件]
        IE --> OUTBOX[(Outbox表)]
        OUTBOX --> MQ[消息队列]
        MQ --> RS[远程服务]
    end
    
    %% 事务边界
    subgraph "事务边界"
        TX
        OUTBOX
    end
    
    %% 连接
    BD -.同一事务.-> IE
    
    %% 样式
    classDef tx fill:#ffcdd2
    classDef event fill:#c8e6c9
    classDef async fill:#fff9c4
    
    class TX,OUTBOX tx
    class DE,IE event
    class MQ,RS async
```

这些图表清楚地展示了 NetCorePal 框架如何通过 CAP 组件实现最终一致性：

1. **本地事务保证强一致性** - 业务数据和集成事件在同一事务中提交
2. **Outbox模式确保可靠性** - 事件持久化后异步发布
3. **消息队列保证投递** - 支持重试和错误处理
4. **远程服务最终一致** - 通过集成事件同步状态