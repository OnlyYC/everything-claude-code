---
name: springboot-verification
description: Spring Boot 项目验证流程：构建、静态分析、测试覆盖率、安全扫描、差异审查。用于发版前或提交 PR 前的完整检查。
---

# Spring Boot 验证流程

在 PR 前、重大变更后、部署前运行。

## 阶段 1：构建

```bash
mvn -T 4 clean verify -DskipTests
```

如果构建失败，停止并修复。

## 阶段 2：静态分析

```bash
# Maven 常用插件
mvn -T 4 spotbugs:check checkstyle:check pmd:check

# 或使用组合命令
mvn -T 4 clean verify
```

## 阶段 3：测试 + 覆盖率

```bash
# 运行测试
mvn -T 4 test

# 生成覆盖率报告
mvn jacoco:report

# 检查覆盖率是否达标（通常要求 80%+）
```

报告内容：
- 总测试数、通过/失败数
- 覆盖率百分比（行覆盖率/分支覆盖率）

## 阶段 4：安全扫描

```bash
# 依赖漏洞检测
mvn org.owasp:dependency-check-maven:check

# 密钥检测（git）
git secrets --scan  # 如果配置了的话
```

## 阶段 5：代码格式检查（可选）

```bash
# 使用 Spotless 插件
mvn spotless:check

# 自动格式化
mvn spotless:apply
```

## 阶段 6：差异审查

```bash
# 查看变更统计
git diff --stat

# 查看详细差异
git diff
```

检查清单：
- 没有遗留调试日志（`System.out`、无守卫的 `log.debug`）
- 有意义的错误信息和 HTTP 状态码
- 必要的地方添加了事务和验证
- 配置变更已记录

## 输出模板

```
验证报告
===================
构建：     [通过/失败]
静态分析： [通过/失败] (spotbugs/checkstyle/pmd)
测试：     [通过/失败] (X/Y 通过，Z% 覆盖率)
安全：     [通过/失败] (发现 N 个漏洞)
差异：     [X 个文件变更]

整体：     [就绪 / 未就绪]

待修复问题：
1. ...
2. ...
```

## 持续模式

- 在重大变更或长会话期间每 30-60 分钟重新运行各阶段
- 保持快速循环：`mvn -T 4 test` + spotbugs 快速反馈

**记住**：快速反馈胜过后期意外。严格把关——在生产系统中将警告视为缺陷。
