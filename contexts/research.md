# 调研上下文

模式：探索、调研、学习
重点：理解先行，行动在后

## 行为准则
- 广泛阅读后再下结论
- 主动提出澄清性问题
- 边调研边记录发现
- 理解清晰前不写代码

## 调研流程
1. 明确调研目标
2. 探索相关代码/文档
3. 形成技术假设
4. 验证假设（查源码/跑测试）
5. 总结调研结论

## 技术栈调研重点

### Spring Boot 3 生态
- **官方文档优先**：Spring Framework Reference、Spring Boot Documentation
- **源码分析**：关键注解实现（@RestController、@Service、@Mapper）
- **版本兼容**：Java 17+ 要求、Jakarta EE 9+ 命名空间变更

### MyBatis-Plus 调研
- **官方文档**：baomidou.com 中文文档
- **源码理解**：BaseMapper 原理、IService 服务层封装
- **配置调研**：分页插件、代码生成器、字段填充策略

### 数据库设计调研
- **表结构分析**：ER 图、索引设计、字段类型选择
- **查询优化**：Explain 分析、慢查询定位
- **数据迁移**：Flyway/Liquibase 版本管理

## 推荐工具
- Read：阅读源码理解实现
- Grep、Glob：检索代码模式（如查找所有 Controller）
- WebSearch：查询技术方案、Issue 解决方案
- Task with Explore：代码库整体结构调研

## 输出格式
先陈述调研发现，后给出技术建议

## 常见调研场景

### 新功能接入调研
1. 查阅官方文档和最佳实践
2. 搜索社区案例和踩坑经验
3. 分析现有代码是否有类似实现
4. 评估技术方案可行性

### Bug 排查调研
1. 复现问题，收集日志堆栈
2. 定位问题代码（Controller → Service → Mapper）
3. 查阅相关 Issue 和 PR
4. 验证修复方案
