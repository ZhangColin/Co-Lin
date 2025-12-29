# DDD战术设计 - 代码实践模式

## 说明
这是DDD在代码层面的实现模式和约束。战术设计只适用于软件开发领域，是在写代码时的指导性方法。这些模式应该根据团队能力务实选择，不是必须全部采用。

**重要提醒**：战略设计做好了（边界清晰、职责明确），代码层面用CRUD还是用这些复杂模式，都可以。

---

## 一、基础概念

### 统一语言（Ubiquitous Language）

**不是**：
- 业务术语的简单搬运
- 行业标准术语
- 领域专家的行话

**是**：
- 团队（专家+开发者）共同创造的语言
- 在代码中精确表达的语言
- 持续演进的活语言

**实践方法**：
1. 白板画图，标注名词和动作
2. 词汇表记录术语和定义
3. 团队对话中使用
4. 最重要：**在代码中表达**

### 贫血模型的陷阱

**贫血症状**：
- 大量getter/setter
- 业务逻辑在Service层
- 对象只是数据容器

**为什么会贫血**：
- JavaBean/属性的历史影响
- ORM框架的早期限制
- 示例代码的误导
- 数据库思维主导

**治疗方案**：
```java
// 贫血 - 意图不明确
backlogItem.setSprintId(sprintId);
backlogItem.setStatus(COMMITTED);

// 健康 - 业务意图清晰
backlogItem.commitTo(sprint);
```

**核心**：行为 > 数据，意图 > 实现

### Modules（模块）

**目的**：
- 组织领域对象
- 不是技术包装，是概念分组
- 像厨房抽屉：按用途组织，不是按质地

**命名原则**：
- 反映统一语言
- 例：`com.company.agilepm.domain.model.product`
- 不用品牌名，用领域概念

**设计规则**：
- 模块之间尽量无循环依赖（acyclic）
- 父子模块可以放松规则
- 模块是活的，随领域演进而调整
- 先考虑Module，再考虑拆分Bounded Context

---

## 二、Aggregates（聚合）- 最难也最重要

### 什么是Aggregate

- 一致性边界
- 事务边界
- 一组必须保持一致的对象

### 四大核心规则（Rules of Thumb）

#### 规则1：在一致性边界内建模真实不变量

**不变量（Invariant）**：
- 必须始终一致的业务规则
- 例：`c = a + b`，当a=2, b=3时，c必须是5

**原则**：
- 一个事务只修改一个Aggregate实例
- Aggregate = 事务一致性边界
- 边界内的一切在事务提交时必须一致

**虚假不变量的危险**：
- 错误：认为Product必须包含所有BacklogItem才能保持一致
- 正确：识别真实的业务规则，其他的用最终一致性

#### 规则2：设计小聚合

**为什么要小**：
- 大聚合 = 并发冲突
- 大聚合 = 性能差
- 大聚合 = 不可扩展

**多小算小**：
- 70%的Aggregates：只有Root Entity + 若干Value Objects
- 30%的Aggregates：Root + 2-3个Entities
- 原则：只包含必须一致的属性，不多不少

**例子分析**：
```java
// 大聚合 - 并发灾难
Product {
  Set<BacklogItem> backlogItems;  // 可能有数千个
  Set<Release> releases;
  Set<Sprint> sprints;
}
// 结果：Bill添加BacklogItem，Joe添加Release → Joe的提交失败！

// 小聚合 - 各自独立
Product { ProductId, name, description }
BacklogItem { BacklogItemId, ProductId, ... }
Release { ReleaseId, ProductId, ... }
Sprint { SprintId, ProductId, ... }
```

**优先用Value Objects而非Entity Parts**：
- Value Objects：不可变、更小、更快、更安全
- Entity Parts：需要单独跟踪、更大开销

#### 规则3：用Identity引用其他Aggregates

**不要**：
```java
public class BacklogItem {
  private Product product;  // 直接对象引用
}
```

**要**：
```java
public class BacklogItem {
  private ProductId productId;  // 用ID引用
}
```

**好处**：
1. 更小：不会加载整个关联对象
2. 更快：减少内存和加载时间
3. 防止误用：不能在同一事务中修改两个Aggregates
4. 可分布式：可以跨系统引用

**如何导航**：
- 不要在Aggregate内部用Repository查找
- 在Application Service中提前查找依赖
- 传递给Aggregate的方法

```java
// Application Service负责查找
@Transactional
public void assignTeamMemberToTask(
  String backlogItemId, String taskId, String teamMemberId) {
  
  // 查找两个Aggregates
  BacklogItem backlogItem = backlogItemRepository.find(backlogItemId);
  Team team = teamRepository.find(backlogItem.teamId());
  
  // 只修改一个，另一个只用于查询
  backlogItem.assignTeamMemberToTask(teamMemberId, team, taskId);
}
```

#### 规则4：边界外用最终一致性

**核心思想**：
- 跨Aggregates的规则不要求立即一致
- 通过Domain Events实现最终一致

**判断标准 - "这是谁的工作？"**：
- 如果是**用户的工作** → 事务一致性
- 如果是**系统的工作** → 最终一致性

**例子**：
```java
// BacklogItem发布事件
public class BacklogItem {
  public void commitTo(Sprint sprint) {
    // 业务逻辑...
    
    // 发布事件让其他Aggregate最终一致
    DomainEventPublisher.instance().publish(
      new BacklogItemCommitted(
        this.tenantId(),
        this.backlogItemId(),
        this.sprintId()
      )
    );
  }
}

// Sprint异步监听并响应
public class SprintEventSubscriber {
  public void handle(BacklogItemCommitted event) {
    // 在另一个事务中
    Sprint sprint = sprintRepository.find(event.sprintId());
    sprint.recordCommitment(event.backlogItemId());
  }
}
```

**何时可以延迟**：
- 几秒到几分钟：通常可接受
- 几小时甚至几天：有时也可以
- 询问领域专家：他们往往比开发者更能接受延迟

**技术实现**：
- Domain Events + 消息机制
- 后台任务
- 事件驱动架构

#### 可以打破规则的4种情况

1. **UI便利性**：批量创建多个Aggregate实例
   - 例：一次创建10个BacklogItems
   - 可以，因为是创建独立的Aggregates，不是修改依赖关系

2. **缺乏技术机制**：没有消息/异步处理能力
   - 现实中可能没有消息中间件
   - 必须权衡：保持规则 vs 交付功能

3. **全局事务**：遗留系统强制两阶段提交
   - 至少在自己的Bounded Context内遵守规则
   - 减少本地的事务冲突

4. **查询性能**：有时需要直接引用
   - 可以用直接引用但标记为懒加载
   - 或考虑CQRS、视图优化

### 实现技巧

**Law of Demeter（最少知识原则）**：
- 方法只能调用：自己、参数、自己创建的对象、直接持有的对象
- 不要：`obj.getPart().getSubPart().doSomething()`
- 要：`obj.doSomethingWithSubPart()`

**Tell, Don't Ask（告诉，不要询问）**：
```java
// 不要Ask - 暴露内部状态
if (backlogItem.getStatus() == COMMITTED && 
    backlogItem.getTasks().allHoursZero()) {
  backlogItem.setStatus(DONE);
}

// Tell - 封装业务逻辑
backlogItem.checkIfDone();
```

### 深度案例：BacklogItem和Task的分合抉择

**场景**：BacklogItem是否应该包含Task？

**不变量**：
- 当所有Task的剩余小时数为0时，BacklogItem状态自动变为Done
- 当有Task剩余小时数>0且状态已是Done时，自动回退状态

**分析方法 - BOTE（Back-of-the-Envelope）计算**：
- Sprint通常12天
- 每个Task通常12小时，每天估算一次 = 12次估算
- 每个BacklogItem通常12个Tasks
- 总共：12 Tasks × 12 EstimationLogs = 144个对象

**考虑因素**：
1. **并发冲突**：多个用户会同时修改同一BacklogItem的不同Task吗？
2. **内存开销**：加载144个对象对性能影响多大？
3. **懒加载**：实际只需加载1个Task + 12个估算 = 13个对象
4. **Story Points优化**：经验丰富的团队可能只用1次估算

**决策过程**：
1. 问："这是谁的工作？"
   - 如果是团队成员的工作 → 保持在一起，事务一致
   - 如果是产品负责人审批后才Done → 可以分开

2. 性能测试
3. 用户接受度测试（如果用最终一致性）
4. 保留两手准备

**最终选择**：先保持在一起，原因：
- 并发冲突很少（一个人一个Task）
- 内存开销不大（实际<25个对象）
- 真实不变量需要保护
- 可以后期拆分

**关键洞察**：
- 发现了workflow preference的需求
- 不同团队可能需要不同的自动化程度
- 深入分析30-60分钟，价值巨大

---

## 三、Entities（实体）

**核心特征**：
- 有唯一标识
- 有生命周期
- 可变（但标识不变）
- 关注"是什么"而非"有什么"

**如何发现Entity**：
- 听关键词："change"（改变）、"find"（查找）、"authenticate"（认证）
- 需要在众多对象中找到某一个
- 对象的状态会随时间变化

**构造原则**：
- 构造时至少包含唯一标识
- 如果有不变量（Invariant），构造时必须满足
- 避免无参构造函数（除非是持久化框架需要）

**实现技巧**：
```java
// 不变量保护
public class User {
  private TenantId tenantId;  // 不能为null
  private String username;     // 不能为null且不可改
  
  protected User(TenantId aTenantId, String aUsername, ...) {
    this.setTenantId(aTenantId);  // 检查null
    this.setUsername(aUsername);   // 检查null和不变性
    ...
  }
  
  protected void setUsername(String aUsername) {
    if (this.username != null) {
      throw new IllegalStateException("Username cannot be changed");
    }
    if (aUsername == null) {
      throw new IllegalArgumentException("Username is required");
    }
    this.username = aUsername;
  }
}
```

**角色与职责**：
- Entity可以实现多个接口，扮演不同角色
- 避免对象精神分裂（Object Schizophrenia）
- 优先用组合而非继承

---

## 四、Value Objects（值对象）

**核心特征**：
1. **度量、量化或描述**：不是"事物"本身，而是对事物的描述
2. **不可变**：创建后不能改变，只能替换
3. **概念整体**：属性放在一起才有意义
4. **可替换**：整体替换，不修改属性
5. **值相等**：通过属性比较相等性
6. **无副作用行为**：方法不修改自身状态

**判断是Entity还是Value Object**：
- 关注点在"属性"而非"标识" → Value Object
- 需要"改变"而非"替换" → Entity
- 需要"查找特定某一个" → Entity

**Whole Value（整体值）**：
```java
// 错误 - 分散的属性
public class ThingOfWorth {
  private BigDecimal amount;     // 50,000,000
  private String currency;        // dollars
}

// 正确 - Whole Value
public class ThingOfWorth {
  private MonetaryValue worth;  // {50,000,000 dollars}
}

public final class MonetaryValue {
  private final BigDecimal amount;
  private final Currency currency;
  
  public MonetaryValue(BigDecimal amount, Currency currency) {
    this.amount = amount;
    this.currency = currency;
  }
  
  // Side-Effect-Free行为
  public MonetaryValue add(MonetaryValue other) {
    // 返回新实例，不修改自己
    return new MonetaryValue(
      this.amount.add(other.amount),
      this.currency
    );
  }
}
```

**避免基本类型偏执**：
```java
// 不要
private String name;  // 什么名字？怎么格式化？

// 要
private FullName name;  // 完整名字，封装所有规则

public final class FullName {
  private final String firstName;
  private final String lastName;
  
  public FullName withMiddleInitial(String initial) {
    return new FullName(this.firstName, initial, this.lastName);
  }
}
```

**集成时用Value Objects最小化**：
- 上游Context有复杂的User和Role两个Aggregates
- 下游Context只需创建Moderator这个简单Value Object
- 避免维护完整的Entity状态同步

**Standard Types**：
- 用Value Objects表示类型（如PhoneType: HOME/MOBILE/WORK）
- 可以用enum实现简单的Standard Types
- 复杂的可以有专门的Context管理

---

## 五、Domain Services（领域服务）

**什么时候需要Domain Service**：
1. 执行重要的业务过程
2. 将领域对象从一种组合转换为另一种
3. 计算需要多个领域对象作为输入

**不应该用Domain Service的情况**：
- 操作自然属于某个Entity或Value Object
- 会导致贫血领域模型

**判断方法**：
- 感觉不舒服：这个方法放在哪个Entity都不合适
- 需要协调多个Aggregates
- 跨Aggregate的复杂计算

**设计原则**：
- 无状态（Stateless）
- 接口体现统一语言
- 名字是动词或动词短语
- 输入/输出都是领域对象

**例子：认证服务**：
```java
// 不好 - 客户端需要理解认证逻辑
User user = userRepository.find(tenantId, username);
if (user != null && user.isAuthentic(password)) {
  // 还忘了检查tenant是否active!
}

// 好 - 封装在Domain Service中
public class AuthenticationService {
  public UserDescriptor authenticate(
    TenantId tenantId, 
    String username, 
    String password) {
    
    Tenant tenant = tenantRepository.find(tenantId);
    if (tenant == null || !tenant.isActive()) {
      return null;
    }
    
    User user = userRepository.find(tenantId, username);
    if (user == null || !user.isAuthentic(password)) {
      return null;
    }
    
    return user.toDescriptor();
  }
}
```

**与Application Service的区别**：
- Domain Service：业务逻辑，无状态，领域层
- Application Service：用例编排，事务管理，应用层

**警告**：
- 不要创建"mini-layer"的Domain Services
- 会导致所有业务逻辑都在Service里 = 贫血模型
- 大部分行为应该在Entity和Value Object上

---

## 六、Domain Events（领域事件）

**定义**：
- 领域中发生的、领域专家关心的事情
- 已经发生（过去式）
- 不可变

**如何识别**：
- 听关键词："When..."、"If that happens..."、"Notify me if..."
- 业务规则："当X发生时，Y必须..."
- 跨Aggregate的协调需求

**命名原则**：
```java
// 命令（现在式）
backlogItem.commitTo(sprint);

// 事件（过去式）
BacklogItemCommitted
```

**Event的属性**：
```java
public class BacklogItemCommitted implements DomainEvent {
  private final Date occurredOn;        // 发生时间
  private final TenantId tenantId;      // 多租户标识
  private final BacklogItemId backlogItemId;  // 事件主体
  private final SprintId committedToSprintId; // 相关对象
  
  // 包含足够信息让订阅者处理，无需回查
}
```

**发布-订阅模式**：
```java
// Aggregate发布事件
public class BacklogItem {
  public void commitTo(Sprint sprint) {
    // 业务逻辑
    ...
    
    // 发布事件
    DomainEventPublisher.instance().publish(
      new BacklogItemCommitted(...)
    );
  }
}

// 轻量级本地订阅
DomainEventPublisher.instance()
  .subscribe(new DomainEventSubscriber<BacklogItemCommitted>() {
    public void handleEvent(BacklogItemCommitted event) {
      // 在同一事务中处理
      Sprint sprint = sprintRepository.find(event.sprintId());
      sprint.recordCommitment(event.backlogItemId());
    }
  });
```

**Event的用途**：
1. **实现最终一致性**：跨Aggregate协调
2. **系统集成**：发布到消息队列，通知其他Context
3. **审计日志**：记录所有发生的事
4. **Event Sourcing**：用Events重建Aggregate状态
5. **替代批处理**：实时响应，而非夜间批量

**Event Store**：
- 可以持久化所有Events
- 用于审计、重放、调试
- 可作为消息转发的缓冲

**远程发布考虑**：
- 需要消息中间件（RabbitMQ、Kafka等）
- Event需要唯一标识（去重）
- 订阅者要处理重复消息（幂等性）
- 可能需要XA事务或Event Store中转

---

## 七、Repositories（仓储）

**定义**：
- 提供像集合一样的接口
- 存取Aggregates
- 隐藏持久化细节

**Repository vs DAO**：
- Repository: 领域概念，面向Aggregate
- DAO: 技术概念，面向数据表

**两种设计风格**：

**1. 集合导向（Collection-Oriented）**：
- 模拟Set集合
- 不需要`save()`方法
- 依赖隐式变更跟踪（如Hibernate）

```java
public interface CalendarEntryRepository {
  void add(CalendarEntry entry);
  void remove(CalendarEntry entry);
  CalendarEntry calendarEntryOfId(CalendarEntryId id);
  // 注意：没有save()方法！
}

// 使用
Calendar calendar = repository.findById(id);
calendar.rename("New Name");  // 自动持久化，无需save()
```

**2. 持久化导向（Persistence-Oriented）**：
- 需要显式`save()`方法
- 适用于不支持变更跟踪的存储（如REST、MongoDB）

```java
public interface CalendarEntryRepository {
  void save(CalendarEntry entry);
  void remove(CalendarEntry entry);
  CalendarEntry calendarEntryOfId(CalendarEntryId id);
}

// 使用
Calendar calendar = repository.findById(id);
calendar.rename("New Name");
repository.save(calendar);  // 必须显式save
```

**关键设计决策**：

**每个Aggregate一个Repository**：
- 只有Aggregate Roots有Repository
- Entity Parts通过Root访问

**接口在领域层，实现在基础设施层**：
```
domain/
  model/
    calendar/
      Calendar.java
      CalendarRepository.java  ← 接口
infrastructure/
  persistence/
    HibernateCalendarRepository.java  ← 实现
```

**提供nextIdentity()方法**：
```java
public interface CalendarRepository {
  CalendarId nextIdentity();  // 生成新ID
  void add(Calendar calendar);
  Calendar calendarOfId(CalendarId id);
}

// 使用
CalendarId id = repository.nextIdentity();
Calendar calendar = new Calendar(id, "My Calendar");
repository.add(calendar);
```

**Finder方法命名**：
- 用业务语言命名
- 不要用通用的`findByXXX`
- 例如：`overlappingCalendarEntries()`而非`findByTimeSpan()`

**事务管理**：
- Repository不管理事务
- 事务由Application Service管理
- Repository操作在事务上下文中执行

**查询性能考虑**：
- 简单查询在Repository
- 复杂报表查询考虑CQRS
- 避免N+1查询问题

---

## 八、Factories（工厂）

**何时需要Factory**：
- 创建复杂Aggregate
- 隐藏创建细节
- 重建from data store
- 跨Context创建对象

**三种实现方式**：

**1. Aggregate上的Factory Method**：
```java
public class Product {
  // Factory Method创建关联Aggregate
  public BacklogItem planBacklogItem(
    String summary, 
    String description,
    BacklogItemType type) {
    
    // 创建新的Aggregate
    BacklogItem item = new BacklogItem(
      this.tenantId(),
      this.productId(),
      backlogItemRepository.nextIdentity(),
      summary,
      description,
      type
    );
    
    // 发布事件
    DomainEventPublisher.publish(
      new ProductBacklogItemPlanned(...)
    );
    
    return item;
  }
}
```

**2. Domain Service作为Factory**：
```java
public class CalendarService {
  // 创建需要跨Aggregate协调的对象
  public Calendar createCalendar(
    TenantId tenantId,
    CalendarId calendarId,
    String name,
    String description,
    Owner owner,
    Set<Participant> participants) {
    
    // 可能需要验证owner和participants
    // 可能需要查询其他Aggregates
    
    Calendar calendar = new Calendar(
      tenantId, calendarId, name, description,
      owner, participants
    );
    
    return calendar;
  }
}
```

**3. 专门的Factory类**（较少用）：
```java
public class BacklogItemFactory {
  public BacklogItem create(...) {
    // 复杂的创建逻辑
  }
}
```

**Factory vs Constructor**：
- 简单对象：用Constructor
- 复杂创建逻辑：用Factory Method
- 需要多种创建方式：用Factory
- 创建需要业务规则：用Factory

**重建对象（Reconstitution）**：
- ORM框架负责从数据库重建
- 不要在构造函数中放业务逻辑
- 提供protected构造函数给ORM

```java
public class Product {
  // 业务构造
  public Product(TenantId tenant, ProductId id, String name) {
    // 验证规则
    this.setName(name);  // 可能抛出异常
  }
  
  // ORM重建用
  protected Product() {
    // 空构造，ORM会直接设置字段
  }
}
```

---

## 九、Application Services（应用服务）

**角色定位**：
- 用例编排者（Use Case Coordinator）
- 领域模型的直接客户
- 不包含业务逻辑（那是领域层的事）

**职责**：
1. 事务管理
2. 安全检查
3. 查找Aggregates（通过Repository）
4. 调用领域对象的行为
5. 发布Domain Events（如果需要跨系统）

**设计原则**：
- 薄薄一层，不要变厚
- 不要有if-else业务判断
- 不要有for循环处理集合

```java
@Transactional
public class ProductBacklogItemService {
  
  @Transactional
  public void commitBacklogItemToSprint(
    String tenantId,
    String backlogItemId,
    String sprintId) {
    
    // 1. 查找Aggregates
    BacklogItem backlogItem = 
      backlogItemRepository.find(new BacklogItemId(backlogItemId));
    Sprint sprint = 
      sprintRepository.find(new SprintId(sprintId));
    
    // 2. 执行领域行为（业务逻辑在这里）
    backlogItem.commitTo(sprint);
    
    // 3. 不需要显式save（如果是Collection-Oriented Repository）
  }
}
```

**Application Service vs Domain Service**：
| 维度 | Application Service | Domain Service |
|------|---------------------|----------------|
| 位置 | 应用层 | 领域层 |
| 职责 | 用例编排、事务 | 业务逻辑 |
| 依赖 | 可依赖基础设施 | 只依赖领域 |
| 测试 | 集成测试 | 单元测试 |

---

## 十、Integration（集成限界上下文）

**分布式系统八原则**（必须牢记）：
1. 网络不可靠
2. 总有延迟
3. 带宽有限
4. 网络不安全
5. 拓扑会变
6. 管理员分散
7. 传输有成本
8. 网络是异构的

**三种集成方式**：

**1. RESTful HTTP - Open Host Service**：
```java
// 提供方
@Path("/tenants/{tenantId}/users")
public class UserResource {
  @GET
  @Path("{username}/inRole/{role}")
  public Response getUserInRole(...) {
    User user = accessService.userInRole(tenantId, username, role);
    return user != null ? ok(user) : noContent();
  }
}

// 消费方 - Anticorruption Layer
public class TranslatingCollaboratorService {
  public Author authorFrom(Tenant tenant, String identity) {
    // 1. 调用远程API
    String json = httpClient.get("/users/" + identity + "/inRole/Author");
    
    // 2. 翻译成本地模型
    return translator.toAuthor(json);
  }
}
```

**优点**：简单、通用、易理解
**缺点**：同步调用、耦合、下游依赖上游可用性

**2. Messaging - 发布-订阅**：
```java
// 发布方
public class Role {
  public void assignUser(User user) {
    // 业务逻辑...
    
    // 发布事件
    DomainEventPublisher.publish(
      new UserAssignedToRole(
        tenantId, 
        user.username(), 
        this.name()
      )
    );
  }
}

// 订阅方
public class ProductOwnerEnablerSubscriber {
  public void handleEvent(UserAssignedToRole event) {
    if (event.roleName().equals("ScrumProductOwner")) {
      // 创建本地的ProductOwner
      productOwnerRepository.add(
        new ProductOwner(event.tenantId(), event.username())
      );
    }
  }
}
```

**优点**：异步、解耦、高可用
**缺点**：最终一致性、复杂度高

**3. RPC（不推荐）**：
- 伪装成本地调用
- 容易忽略网络问题
- 级联失败

**Anticorruption Layer（防腐层）模式**：

**三件套**：
1. **Service接口**：定义本地需要的操作
2. **Adapter**：调用远程系统
3. **Translator**：翻译外部模型到本地模型

```
远程Context → [Adapter] → [Translator] → 本地模型
                    ↓
               Service接口（隐藏细节）
```

**为什么需要ACL**：
- 保护本地模型不被污染
- 外部变化不影响本地
- 用本地统一语言

**Published Language（发布语言）**：
- 定义数据交换格式（JSON/XML/Protobuf）
- 定义Custom Media Type
- 形成约定合同

```java
// Notification媒体类型规范
{
  "notificationId": 123,
  "typeName": "com.company.BacklogItemCommitted",
  "version": 1,
  "occurredOn": "2025-12-29T10:00:00Z",
  "event": {
    "backlogItemId": {"id": "abc"},
    "sprintId": {"id": "xyz"},
    "tenantId": {"id": "tenant1"}
  }
}

// 不共享类，用NotificationReader读取
NotificationReader reader = new NotificationReader(json);
String backlogItemId = reader.eventStringValue("backlogItemId.id");
String sprintId = reader.eventStringValue("sprintId.id");
```

**集成策略选择**：
- **查询操作**：REST（同步获取数据）
- **状态同步**：Messaging（异步最终一致）
- **实时协作**：考虑两者结合

**何时用Value Object集成**：
- 上游有复杂Aggregate（User + Role）
- 下游只需要简单标识（Moderator）
- 不需要保持同步
- 最小化集成

---

## 十一、技术关注点提醒

**不要在领域对象中混入技术关注点**：
- 错误：在业务方法中检查权限
- 正确：在Application Service处理技术关注点

**领域层应该保持纯粹**：
- 只关注业务逻辑
- 不依赖具体的技术框架
- 便于测试和演进

---

## 十二、战术设计检查清单

- [ ] 统一语言在代码中清晰表达了吗？
- [ ] 领域对象有行为，不只是数据容器？
- [ ] 业务规则在领域对象中，不在Service？
- [ ] Aggregate边界合理吗（小聚合原则）？
- [ ] 用ID引用其他Aggregates了吗？
- [ ] 跨Aggregate用最终一致性了吗？
- [ ] 测试展示了如何使用模型吗？
- [ ] 领域专家能理解代码表达的概念吗？

---

**最后的话**：

战术设计的模式和约束，**根据团队能力务实选择**。

关键是：
1. 代码清晰表达业务意图
2. 业务逻辑不分散在各处
3. 便于测试和维护

如果战略设计做好了（边界清晰、职责明确），代码层面用简单的CRUD实现，也完全可以。不要为了DDD而DDD。

