---
name: coding-standards
description: 通用代码品质原则与最佳实践。适配 Java 21 + Spring Boot 3 + MyBatis-Plus + MySQL 技术栈。专注通用原则，不包含特定语言语法细节。
version: 2.0.0
related_skills: [java-coding-standards, java-patterns, springboot-patterns, java-testing]
---

# 代码标准与最佳实践

通用代码品质原则，适用于所有编程语言。

## 本技能定位

本技能聚焦**通用代码品质原则**，不包含特定语言语法细节：
- **通用原则**：可读性、KISS、DRY、YAGNI 等
- **设计原则**：SOLID、设计模式、架构原则
- **质量标准**：测试、文档、性能、安全

**Java 特定规范**请参考：
- `java-coding-standards` - Java 21 语法特性、命名规范
- `java-patterns` - Alibaba Java 开发手册
- `springboot-patterns` - Spring Boot 架构模式

## 代码品质原则

### 1. 可读性优先

**核心思想**：代码被阅读的次数远多于被撰写的次数

```
良好示例：清晰的变量命名
const maxRetryCount = 3
const userAuthenticationToken = "abc123"
const databaseConnectionPoolSize = 20

不良示例：模糊的命名
const n = 3
const t = "abc123"
const s = 20
```

**实践要点**：
- 使用描述性名称，不使用缩写
- 函数名称表达意图（动词-名词模式）
- 保持一致的代码格式
- 优先自文档化代码，而非注释

### 2. KISS（Keep It Simple, Stupid）

**核心思想**：简单方案优于复杂方案

```
良好示例：直接表达
function isActive(user) {
    return user.status === 'active'
}

不良示例：过度复杂
function isActive(user) {
    const status = user.status || 'unknown'
    const activeStates = ['active', 'enabled', 'online']
    return activeStates.includes(status) && status !== 'disabled'
}
```

**实践要点**：
- 使用最简单的解决方案
- 避免过度工程
- 不做过早优化
- 易于理解 > 聪明的代码

### 3. DRY（Don't Repeat Yourself）

**核心思想**：避免重复代码

```
良好示例：提取公共逻辑
function validateEmail(email) {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)
}

function validateUsername(username) {
    return username.length >= 3 && username.length <= 20
}

function validateUser(user) {
    return validateEmail(user.email) && validateUsername(user.username)
}

不良示例：重复验证逻辑
function validateUser1(user) {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
    if (!emailRegex.test(user.email)) return false
    if (user.username.length < 3 || user.username.length > 20) return false
    return true
}
```

**实践要点**：
- 将共用逻辑提取为函数/方法
- 建立可重用的组件
- 在模块间共享工具函数
- 避免复制粘贴程序设计

### 4. YAGNI（You Aren't Gonna Need It）

**核心思想**：只实现当前需要的功能

```
良好示例：按需实现
class UserValidator {
    validateEmail(email) { /* 实现 */ }
    validateUsername(username) { /* 实现 */ }
}

不良示例：过度设计
class UserValidator {
    validateEmail(email) { /* 实现 */ }
    validateUsername(username) { /* 实现 */ }
    validatePhone(phone) { /* 当前不需要 */ }
    validateAddress(address) { /* 当前不需要 */ }
    validateSSN(ssn) { /* 可能永远不需要 */ }
}
```

**实践要点**：
- 在需要之前不要构建功能
- 避免推测性的通用化
- 只在需要时增加复杂度
- 从简单开始，需要时再重构

## SOLID 设计原则

### S - 单一职责原则

每个类/模块应该只有一个改变的理由。

```
良好示例：职责分离
class UserService {
    createUser(user) { /* 用户创建逻辑 */ }
    updateUser(user) { /* 用户更新逻辑 */ }
}

class EmailService {
    sendWelcomeEmail(user) { /* 发送欢迎邮件 */ }
    sendNotification(user) { /* 发送通知 */ }
}

不良示例：职责混乱
class UserService {
    createUser(user) { /* 用户创建 */ }
    sendEmail(user) { /* 发送邮件 */ }
    logActivity(user) { /* 记录日志 */ }
    generateReport(user) { /* 生成报表 */ }
}
```

### O - 开闭原则

对扩展开放，对修改关闭。

```
良好示例：使用策略模式
interface PaymentStrategy {
    process(amount)
}

class CreditCardPayment {
    process(amount) { /* 信用卡支付 */ }
}

class PayPalPayment {
    process(amount) { /* PayPal 支付 */ }
}

// 添加新支付方式无需修改现有代码
class WeChatPayment {
    process(amount) { /* 微信支付 */ }
}
```

### L - 里氏替换原则

子类可以替换父类而不破坏程序正确性。

```
良好示例：正确的继承关系
class Rectangle {
    area() { return width * height }
}

class Square extends Rectangle {
    // Square 是特殊的 Rectangle，可以安全替换
}

不良示例：违反里氏替换原则
class Bird {
    fly() { /* 飞行 */ }
}

class Penguin extends Bird {
    fly() { throw new Error("企鹅不会飞") }
    // Penguin 不能替换 Bird
}
```

### I - 接口隔离原则

客户端不应该依赖它不需要的接口。

```
良好示例：分离接口
interface Reader {
    read()
}

interface Writer {
    write()
}

// 客户端只依赖需要的接口
class Document implements Reader, Writer {
    read() { /* 实现 */ }
    write() { /* 实现 */ }
}

不良示例：臃肿接口
interface DocumentHandler {
    read()
    write()
    print()
    scan()
    email()
}
```

### D - 依赖倒置原则

高层模块不应该依赖低层模块，都应该依赖抽象。

```
良好示例：依赖抽象
interface Database {
    save(data)
}

class MySQLDatabase implements Database {
    save(data) { /* MySQL 实现 */ }
}

class UserService {
    constructor(database) {
        this.database = database  // 依赖抽象
    }
}

不良示例：依赖具体实现
class UserService {
    constructor() {
        this.database = new MySQLDatabase()  // 依赖具体实现
    }
}
```

## 设计模式

### 1. 策略模式（Strategy）

定义一系列算法，将每个算法封装起来，并使它们可以互换。

```
应用场景：多种支付方式、多种排序算法、多种验证规则

示例代码：
interface PaymentStrategy {
    process(amount)
}

class CreditCardStrategy {
    process(amount) { /* 信用卡处理 */ }
}

class AlipayStrategy {
    process(amount) { /* 支付宝处理 */ }
}

class PaymentContext {
    setStrategy(strategy) {
        this.strategy = strategy
    }
    processPayment(amount) {
        this.strategy.process(amount)
    }
}
```

### 2. 工厂模式（Factory）

定义创建对象的接口，让子类决定实例化哪个类。

```
应用场景：数据库连接池、日志框架、服务实例创建

示例代码：
class DatabaseFactory {
    createConnection(type) {
        switch (type) {
            case 'mysql': return new MySQLConnection()
            case 'postgresql': return new PostgreSQLConnection()
            case 'mongodb': return new MongoDBConnection()
            default: throw new Error('Unsupported database')
        }
    }
}
```

### 3. 观察者模式（Observer）

定义对象间的一对多依赖关系，当一个对象状态改变时，所有依赖者都会收到通知。

```
应用场景：事件系统、消息队列、UI 更新

示例代码：
class EventEmitter {
    constructor() {
        this.listeners = {}
    }

    on(event, callback) {
        if (!this.listeners[event]) {
            this.listeners[event] = []
        }
        this.listeners[event].push(callback)
    }

    emit(event, data) {
        if (this.listeners[event]) {
            this.listeners[event].forEach(cb => cb(data))
        }
    }
}
```

### 4. 单例模式（Singleton）

确保一个类只有一个实例，并提供全局访问点。

```
应用场景：配置管理器、连接池、缓存服务

注意：单例模式可能带来测试问题，优先考虑依赖注入

示例代码：
class ConfigManager {
    constructor() {
        if (ConfigManager.instance) {
            return ConfigManager.instance
        }
        this.config = this.loadConfig()
        ConfigManager.instance = this
    }

    static getInstance() {
        if (!ConfigManager.instance) {
            ConfigManager.instance = new ConfigManager()
        }
        return ConfigManager.instance
    }
}
```

## 代码异味检测

### 1. 过长方法

**问题**：方法超过 50 行，难以理解和维护

```
不良示例：
function processOrderData() {
    // 100 行代码
}

良好示例：
function processOrderData() {
    const data = collectOrderData()
    const transformed = transformOrderData(data)
    saveOrderData(transformed)
}
```

### 2. 过深嵌套

**问题**：嵌套超过 3 层，难以阅读

```
不良示例：
if (user != null) {
    if (user.isAdmin()) {
        if (market != null) {
            if (market.isActive()) {
                if (hasPermission()) {
                    // 做某事
                }
            }
        }
    }
}

良好示例：提前返回
if (user == null) return
if (!user.isAdmin()) return
if (market == null) return
if (!market.isActive()) return
if (!hasPermission()) return

// 做某事
```

### 3. 魔术数字

**问题**：无解释的数字

```
不良示例：
if (retryCount > 3) { }
Thread.sleep(500);

良好示例：
const MAX_RETRY_COUNT = 3
const DEBOUNCE_DELAY_MS = 500

if (retryCount > MAX_RETRY_COUNT) { }
Thread.sleep(DEBOUNCE_DELAY_MS);
```

### 4. 上帝类

**问题**：类承担过多职责

```
不良示例：
class OrderManager {
    createOrder() { }
    sendEmail() { }
    generateReport() { }
    processPayment() { }
}

良好示例：
class OrderService { }
class NotificationService { }
class ReportService { }
class PaymentService { }
```

### 5. 循环复杂度过高

**问题**：复杂的嵌套逻辑

```
不良示例：多层嵌套循环
for (order of orders) {
    for (item of order.items) {
        for (product of products) {
            if (order.id === product.orderId) {
                // 复杂逻辑
            }
        }
    }
}

良好示例：使用 Map 优化
const productMap = new Map(products.map(p => [p.id, p]))

for (const order of orders) {
    for (const item of order.items) {
        const product = productMap.get(item.productId)
        // 处理逻辑
    }
}
```

## 测试标准

### 测试金字塔

```
         /\
        /  \
       / E2E \       - 少量端到端测试
      /______\
     /        \
    / 集成测试 \      - 适中集成测试
   /______________\
  /    单元测试    \   - 大量单元测试
 /__________________\
```

### AAA 模式

每个测试应该遵循 AAA 结构：

```
良好示例：
test('计算折扣价格', () => {
    // Arrange（准备）
    const originalPrice = 100
    const discount = 0.2

    // Act（执行）
    const finalPrice = calculateDiscount(originalPrice, discount)

    // Assert（断言）
    expect(finalPrice).toBe(80)
})
```

### 测试命名

```
良好示例：
test('当用户不存在时抛出异常', () => { })
test('OpenAI API key 缺失时抛出异常', () => { })
test('Redis 不可用时回退到子串搜索', () => { })

不良示例：
test('test', () => { })
test('testSearch', () => { })
```

## 异常处理最佳实践

### 统一异常处理

```
良好示例：统一异常处理器
class GlobalExceptionHandler {
    handleBusinessError(error) {
        return {
            code: error.code,
            message: error.message
        }
    }

    handleSystemError(error) {
        logger.error('System error', error)
        return {
            code: 'SYSTEM_ERROR',
            message: '系统错误，请稍后重试'
        }
    }
}

不良示例：每个方法单独处理
function getMarket(id) {
    try {
        return marketService.getById(id)
    } catch (error) {
        if (error instanceof BusinessError) {
            return fail(error.code, error.message)
        }
        return fail('SYSTEM_ERROR', '系统错误')
    }
}
```

### 异常分类

```
业务异常（Business Exception）：
- 用户输入错误
- 业务规则违反
- 资源不存在

系统异常（System Exception）：
- 数据库连接失败
- 第三方服务超时
- 系统配置错误

处理原则：
- 业务异常：返回用户友好的错误信息
- 系统异常：记录详细日志，返回通用错误信息
```

## 项目结构规范

### 分层架构

```
应用层（Application Layer）
├── 控制器（Controller）- 处理 HTTP 请求
├── DTO（Data Transfer Object）- 数据传输对象
└── 异常处理器（ExceptionHandler）- 统一错误处理

业务层（Business Layer）
├── 服务（Service）- 业务逻辑
├── 领域模型（Domain Model）- 业务实体
└── 业务规则（Business Rules）- 业务验证

数据层（Data Layer）
├── 数据访问对象（DAO/Repository）- 数据库操作
├── 实体（Entity）- 数据库表映射
└── 数据库迁移（Migration）- Schema 版本控制
```

### 依赖规则

```
┌─────────────────────────────────────┐
│          控制器层                  │
│  只依赖业务层，不直接访问数据层      │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│          业务层                      │
│  封装业务逻辑，协调多个数据访问       │
└─────────────────────────────────────┘
              ↓
┌─────────────────────────────────────┐
│          数据层                      │
│  封装数据访问，提供 CRUD 操作        │
└─────────────────────────────────────┘
```

## 性能优化原则

### 1. 过早优化是万恶之源

```
原则：先让代码正确，再让代码快速

良好实践：
1. 实现功能 → 测试验证 → 性能分析 → 针对性优化
2. 使用性能分析工具识别瓶颈
3. 优化前先测量，优化后再验证

不良实践：
- 基于猜测进行优化
- 过度使用缓存
- 过早引入复杂架构
```

### 2. 数据库查询优化

```
常见问题：
- N+1 查询
- 返回过多字段
- 缺少索引
- 大事务

优化策略：
- 批量查询替代循环查询
- 只查询需要的字段
- 添加适当的索引
- 拆分大事务
```

### 3. 缓存策略

```
缓存层次：
L1：应用内存缓存（快速、容量小）
L2：分布式缓存（中等速度、容量大）
L3：数据库（慢速、容量最大）

缓存原则：
- 只缓存热点数据
- 设置合理的过期时间
- 处理缓存失效
- 避免缓存雪崩
```

## 文档规范

### 代码注释

```
良好注释示例：
// 使用指数退避避免 API 服务过载
const delay = Math.min(1000 * Math.pow(2, retryCount), 30000)

// 不需要注释的示例：
const userCount = users.length  // 显而易见，不需要注释
```

### API 文档

```
必要的 API 文档要素：
1. 功能描述
2. 请求参数说明
3. 响应格式说明
4. 错误码说明
5. 使用示例
6. 版本变更记录
```

## 相关技能

- `java-coding-standards` - Java 21 语法特性、命名规范
- `java-patterns` - Alibaba Java 开发手册
- `java-testing` - JUnit 5 + Mockito 测试
- `springboot-patterns` - Spring Boot 架构模式
- `security-review` - 安全审查清单
- `tdd-workflow` - 测试驱动开发流程

---

**记住**：代码品质是不可协商的。遵循通用代码品质原则，保持代码简洁、清晰、可维护。清晰、可维护的代码能实现快速开发和自信的重构。

优先考虑可维护性而非微优化。在证明必要之前，保持简单。
