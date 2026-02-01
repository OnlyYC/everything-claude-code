# 更新代码地图

分析代码库结构并更新架构文档：

1. 扫描所有源文件的 imports、类依赖和注解
2. 以下列格式生成精简的代码地图：
   - `codemaps/architecture.md` - 整体架构（DDD 分层、模块划分）
   - `codemaps/controller.md` - Controller 层结构（API 接口）
   - `codemaps/service.md` - Service 层结构（业务逻辑）
   - `codemaps/mapper.md` - Mapper/DAO 层结构（数据访问）
   - `codemaps/entity.md` - 实体类和数据模型

3. 计算与前一版本的差异百分比
4. 如果变更 > 30%，在更新前请求用户批准
5. 为每个代码地图新增新鲜度时间戳
6. 将报告保存到 `.reports/codemap-diff.txt`

## Java 代码地图结构

### 标准三层架构

```
src/main/java/com/example/
├── controller/          # 控制层
│   ├── UserController.java
│   ├── OrderController.java
│   └── vo/              # 视图对象
│       ├── UserVO.java
│       └── UserQueryVO.java
├── service/             # 服务层
│   ├── UserService.java
│   ├── impl/
│   │   └── UserServiceImpl.java
│   └── dto/             # 数据传输对象
│       ├── UserDTO.java
│       └── UserCreateDTO.java
├── mapper/              # 持久层
│   ├── UserMapper.java
│   └── xml/
│       └── UserMapper.xml
├── entity/              # 实体类
│   ├── User.java
│   └── UserDO.java
├── config/              # 配置类
│   ├── MybatisConfig.java
│   └── WebConfig.java
├── common/              # 通用组件
│   ├── exception/       # 异常处理
│   ├── enums/           # 枚举
│   └── constants/       # 常量
└── util/                # 工具类
```

### DDD 分层架构

```
src/main/java/com/example/
├── interfaces/          # 接口层
│   ├── controller/
│   ├── facade/
│   └── dto/
├── application/         # 应用层
│   ├── service/
│   ├── command/
│   └── query/
├── domain/              # 领域层
│   ├── model/
│   │   ├── aggregate/   # 聚合根
│   │   ├── entity/      # 实体
│   │   └── valueobject/ # 值对象
│   ├── repository/      # 仓储接口
│   └── service/         # 领域服务
└── infrastructure/      # 基础设施层
    ├── persistence/     # 持久化实现
    ├── cache/           # 缓存实现
    └── mq/              # 消息队列
```

## 代码地图模板

### Controller 层地图

```markdown
# Controller 层代码地图

## 用户模块
| Controller | 路径 | 方法 | 说明 |
|-----------|------|------|------|
| UserController | /api/users | GET / | 查询用户列表 |
| UserController | /api/users | POST / | 创建用户 |
| UserController | /api/users/{id} | GET /{id} | 查询用户详情 |

## 依赖关系
- UserController -> UserService
- UserController -> UserVO
```

### Service 层地图

```markdown
# Service 层代码地图

## 用户服务
| Service | 方法 | 依赖 | 说明 |
|---------|------|------|------|
| UserService | getUserById() | UserMapper | 根据 ID 查询 |
| UserService | createUser() | UserMapper | 创建用户 |
| UserService | updateUser() | UserMapper | 更新用户 |

## 依赖关系
- UserService -> UserMapper
- UserService -> UserDTO
```

## 使用 Java 反射分析

使用 Java Reflection API 或 ASM 字节码分析工具来扫描类结构。

专注于高层结构，而非实现细节。
