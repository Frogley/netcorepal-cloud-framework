# NetCorePal 框架最终一致性保证机制回答

## 问题回答

### 这个框架中使用什么包来发送集成事件？

NetCorePal 框架使用 **DotNetCore.CAP** 作为核心包来发送集成事件。具体包括：

1. **DotNetCore.CAP** - 主要的分布式事务和事件处理包
2. **DotNetCore.CAP.SqlServer/MySQL/PostgreSQL** - 数据库特定的 CAP 适配器
3. **DotNetCore.CAP.RabbitMQ** - RabbitMQ 消息队列适配器
4. **MediatR** - 用于本地领域事件处理

### 如何保证微服务间的最终一致性？

框架通过以下机制保证微服务间的最终一致性：

#### 1. Outbox 模式实现
- 集成事件与业务数据在同一个数据库事务中持久化
- 使用 CAP 的 Outbox 表存储待发布的事件
- 确保业务操作和事件发布的原子性

#### 2. 两阶段提交流程
```
阶段一：本地事务
1. 开始数据库事务
2. 执行业务逻辑
3. 触发领域事件
4. 领域事件处理器发布集成事件到 Outbox
5. 提交本地事务

阶段二：异步事件发布
1. CAP 后台服务扫描 Outbox 表
2. 将事件发布到消息队列（RabbitMQ）
3. 远程服务消费事件并处理
4. 支持失败重试（默认10次）
```

#### 3. 核心组件协作

**事件发布端：**
- `CapIntegrationEventPublisher` - 负责将集成事件发布到 CAP
- `IPublisherTransactionHandler` - 协调数据库事务与 CAP 事务
- `AppDbContextBase` - 提供统一的工作单元模式

**事件消费端：**
- `IIntegrationEventHandler<T>` - 集成事件处理器接口
- `[CapSubscribe]` - CAP 订阅器属性
- 自动重试和错误处理机制

#### 4. 关键代码实现

**事务协调（以 SQL Server 为例）：**
```csharp
public class CapSqlServerPublisherTransactionHandler : IPublisherTransactionHandler
{
    public IDbContextTransaction BeginTransaction(DbContext context)
    {
        // 将 CAP 发布器绑定到数据库事务
        return context.Database.BeginTransaction(_capBus.Value, autoCommit: false);
    }
}
```

**集成事件发布：**
```csharp
public sealed class CapIntegrationEventPublisher : IntegrationEventPublisher
{
    protected override Task PublishAsync(IntegrationEventPublishContext context)
    {
        // 在当前数据库事务中发布事件到 Outbox
        return _capPublisher.PublishAsync(
            name: context.Data.GetType().Name,
            contentObj: context.Data,
            headers: context.Headers,
            cancellationToken: context.CancellationToken);
    }
}
```

#### 5. 一致性保证机制

**强一致性（本地服务内）：**
- 业务操作和领域事件处理在同一事务中
- 集成事件存储到 Outbox 与业务数据原子性提交

**最终一致性（跨服务）：**
- CAP 保证已持久化的事件最终会被投递
- 消费端支持重试机制处理临时故障
- 事件处理器需要设计为幂等的

**可靠性保证：**
- 事件重复投递保护（幂等性）
- 死信队列处理无法处理的事件
- 完整的监控和可观测性支持

## 总结

NetCorePal 框架通过 **DotNetCore.CAP** 实现了基于 Outbox 模式的最终一致性保证机制：

1. **本地事务内保证强一致性** - 业务数据与事件在同一事务中提交
2. **异步事件发布保证最终一致性** - 通过消息队列可靠投递事件
3. **重试机制处理临时故障** - 确保事件最终被成功处理
4. **幂等性设计避免重复处理** - 处理器需要支持重复消息

这种设计既保证了单个服务的数据一致性，又实现了分布式系统中服务间的最终一致性，是微服务架构中处理分布式事务的标准解决方案。