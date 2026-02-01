# 测试覆盖率

分析测试覆盖率并生成缺失的测试：

1. 执行带覆盖率的测试：`mvn test jacoco:report`

2. 分析覆盖率报告（target/site/jacoco/index.html）

3. 识别低于 80% 覆盖率阈值的类和方法

4. 对每个覆盖不足的类：
   - 分析未测试的代码路径
   - 为 Service 方法生成单元测试
   - 为 API 接口生成集成测试
   - 为关键流程生成端到端测试

5. 验证新测试通过

6. 显示前后覆盖率指标

7. 确保项目达到 80% 以上整体覆盖率

## 覆盖率工具

```bash
# Maven 命令
mvn test jacoco:report

# 查看 HTML 报告
open target/site/jacoco/index.html

# 按方法查看覆盖率
mvn test jacoco:report && cat target/site/jacoco/jacoco.csv

# 检查覆盖率是否达标
mvn verify jacoco:check
```

## 覆盖率目标

| 代码类型 | 目标覆盖率 |
|---------|-----------|
| 核心业务逻辑 | 100% |
| Service 层 | 90%+ |
| Controller 层 | 80%+ |
| Mapper/DAO 层 | 70%+ |
| 实体类/DTO | 60%+ |
| 配置类 | 排除 |

## 覆盖率配置

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
        <execution>
            <id>check</id>
            <goals>
                <goal>check</goal>
            </goals>
            <configuration>
                <rules>
                    <rule>
                        <element>PACKAGE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

## 排除不需要测试的代码

```java
// 排除 Lombok 生成的代码
@Generated
@Data
@Entity
public class User {
    // ...
}

// 排除配置类
@Configuration
public class WebConfig implements WebMvcConfigurer {
    // ...
}
```

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <configuration>
        <excludes>
            <exclude>**/config/**</exclude>
            <exclude>**/dto/**</exclude>
            <exclude>**/entity/**</exclude>
            <exclude>**/vo/**</exclude>
            <exclude>**/*Config.*</exclude>
        </excludes>
    </configuration>
</plugin>
```

## 覆盖率报告解读

- **指令覆盖率**：执行的字节码指令比例
- **分支覆盖率**：if/switch 分支覆盖比例
- **行覆盖率**：执行的代码行比例
- **方法覆盖率**：被执行的方法比例
- **类覆盖率**：被实例化的类比例

## 覆盖率提升策略

1. **未覆盖的分支**：添加不同参数的测试用例
2. **未覆盖的异常**：添加异常场景测试
3. **未覆盖的方法**：为新方法添加单元测试
4. **未覆盖的类**：为新类添加测试套件

专注于：
- 正常流程场景
- 异常处理流程
- 边界情况（null、空集合、边界值）
- 并发场景
