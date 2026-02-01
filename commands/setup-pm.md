---
description: Configure your preferred build tool (Maven)
disable-model-invocation: true
---

# 构建工具配置

为此项目或全局配置您首选的构建工具。

## 使用方法

```bash
# 检测当前的构建工具
java -jar scripts/build-tool-detector.jar --detect

# 设置全局偏好
java -jar scripts/build-tool-detector.jar --global maven

# 设置项目偏好
java -jar scripts/build-tool-detector.jar --project maven

# 列出可用的构建工具
java -jar scripts/build-tool-detector.jar --list
```

## 检测优先级

决定使用哪个构建工具时，按以下顺序检查：

1. **环境变量**：`CLAUDE_BUILD_TOOL`
2. **项目配置**：`.claude/build-tool.json`
3. **项目文件**：是否存在 pom.xml
4. **全局配置**：`~/.claude/build-tool.json`
5. **回退**：第一个可用的构建工具（Maven）

## 配置文件

### 全局配置
```json
// ~/.claude/build-tool.json
{
  "buildTool": "maven"
}
```

### 项目配置
```json
// .claude/build-tool.json
{
  "buildTool": "maven"
}
```

## 环境变量

设置 `CLAUDE_BUILD_TOOL` 以覆盖所有其他检测方法：

```bash
# Windows (PowerShell)
$env:CLAUDE_BUILD_TOOL = "maven"

# macOS/Linux
export CLAUDE_BUILD_TOOL=maven
```

## Maven 常用命令

| 命令 | 说明 |
|------|------|
| `mvn clean compile` | 清理并编译 |
| `mvn clean package` | 打包 |
| `mvn clean install` | 安装到本地仓库 |
| `mvn test` | 执行单元测试 |
| `mvn verify` | 执行集成测试 |
| `mvn spring-boot:run` | 启动应用 |


## 执行检测

要查看当前构建工具检测结果，执行：

```bash
java -jar scripts/build-tool-detector.jar --detect
```
