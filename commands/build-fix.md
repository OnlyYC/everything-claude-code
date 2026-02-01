# 构建与修复

增量修复 Java 编译和构建错误：

1. 执行构建：`mvn clean compile` 或 `gradle build`

2. 解析错误输出：
   - 按文件分组
   - 按严重性排序（编译错误 > 警告）

3. 对每个错误：
   - 显示错误上下文（前后 5 行）
   - 解释问题
   - 提出修复方案
   - 应用修复
   - 重新执行构建
   - 验证错误已解决

4. 停止条件：
   - 修复引入新错误
   - 3 次尝试后同样错误仍存在
   - 使用者要求暂停

5. 显示摘要：
   - 已修复的错误
   - 剩余的错误
   - 新引入的错误

## 常见 Java 编译错误

| 错误 | 典型修复 |
|------|----------|
| `cannot find symbol` | 新增 import 或修正类名拼写 |
| `package X does not exist` | 新增 Maven 依赖或修正包名 |
| `incompatible types` | 类型转换或修正泛型声明 |
| `missing return statement` | 新增 return 语句 |
| `method X in class Y cannot be applied` | 修正方法参数类型或数量 |
| `cyclic inheritance` | 重构类继承关系 |
| `IOException, FileNotFoundException` | 新增 throws 声明或 try-catch |

## 构建命令

```bash
# Maven 构建
mvn clean compile
mvn clean package
mvn clean install

# Gradle 构建
gradle build
gradle compileJava
gradle bootJar

# 跳过测试构建
mvn clean package -DskipTests
```

为了安全，一次修复一个错误！
