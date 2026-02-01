---
name: iterative-retrieval
description: Pattern for progressively refining context retrieval to solve the subagent context problem
---

# 迭代检索模式

解决多 agent 工作流程中的"上下文问题"，其中子 agents 在开始工作之前不知道需要什么上下文。

## 问题

子 agents 以有限上下文产生。它们不知道：
- 哪些文件包含相关代码
- 代码库中存在什么模式
- 项目使用什么术语

标准方法失败：
- **传送所有内容**：超过上下文限制
- **不传送内容**：Agent 缺乏关键资讯
- **猜测需要什么**：经常错误

## 解决方案：迭代检索

一个渐进精炼上下文的 4 阶段循环：

```
┌─────────────────────────────────────────────┐
│                                             │
│   ┌──────────┐      ┌──────────┐            │
│   │ DISPATCH │─────▶│ EVALUATE │            │
│   └──────────┘      └──────────┘            │
│        ▲                  │                 │
│        │                  ▼                 │
│   ┌──────────┐      ┌──────────┐            │
│   │   LOOP   │◀─────│  REFINE  │            │
│   └──────────┘      └──────────┘            │
│                                             │
│        最多 3 个循环，然后继续               │
└─────────────────────────────────────────────┘
```

### 阶段 1：DISPATCH

初始广泛查询以收集候选文件：

```java
package com.example.project.ai.context;

import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.List;
import java.util.Set;

/**
 * 初始检索查询
 */
public record RetrievalQuery(
        Set<String> patterns,           // 文件模式，如 ["**/*Controller.java", "**/*Service.java"]
        Set<String> keywords,           // 搜索关键字
        Set<String> excludes,           // 排除模式
        Set<String> focusAreas          // 特定关注区域
) {
    public static RetrievalQuery fromTask(String taskDescription) {
        return new RetrievalQuery(
            Set.of("**/*.java", "**/*.xml", "**/*.yml"),
            extractKeywords(taskDescription),
            Set.of("**/test/**", "**/*Test.java", "**/*Tests.java"),
            Set.of()
        );
    }

    private static Set<String> extractKeywords(String description) {
        // 从任务描述中提取初始关键字
        return Set.of(description.toLowerCase().split("\\s+"));
    }
}

// 使用示例
public class ContextDispatcher {
    private final CodebaseRetriever retriever;

    public List<SourceFile> dispatch(String task) {
        RetrievalQuery query = RetrievalQuery.fromTask(task);
        return retriever.retrieve(query);
    }
}
```

### 阶段 2：EVALUATE

评估检索内容的相关性：

```java
package com.example.project.ai.context;

import java.util.List;

/**
 * 文件相关性评估结果
 */
public record RelevanceEvaluation(
        SourceFile file,
        double relevance,               // 0.0 - 1.0
        String reason,                  // 相关性说明
        List<String> missingContext     // 识别的缺失上下文
) {
    public boolean isHighRelevance() {
        return relevance >= 0.7;
    }

    public boolean isLowRelevance() {
        return relevance < 0.2;
    }
}

/**
 * 相关性评估器
 */
public class RelevanceEvaluator {

    public List<RelevanceEvaluation> evaluate(
            List<SourceFile> files,
            String task) {

        return files.stream()
                .map(file -> evaluateFile(file, task))
                .toList();
    }

    private RelevanceEvaluation evaluateFile(SourceFile file, String task) {
        double score = calculateRelevanceScore(file, task);
        String reason = explainRelevance(file, task);
        List<String> gaps = identifyGaps(file, task);

        return new RelevanceEvaluation(file, score, reason, gaps);
    }

    private double calculateRelevance(SourceFile file, String task) {
        double score = 0.0;

        // 直接匹配关键字
        if (file.containsAny(taskKeywords(task))) {
            score += 0.4;
        }

        // 包含相关类型定义
        if (hasRelevantTypes(file)) {
            score += 0.3;
        }

        // 包含相关注解（Spring 特定）
        if (hasRelevantAnnotations(file)) {
            score += 0.2;
        }

        // 包含相关模式
        if (matchesPatterns(file, task)) {
            score += 0.1;
        }

        return Math.min(score, 1.0);
    }

    private String explainRelevance(SourceFile file, String task) {
        StringBuilder reason = new StringBuilder();

        if (file.isController()) {
            reason.append("REST 控制器，处理 API 请求。");
        }
        if (file.isService()) {
            reason.append("业务服务层，包含核心逻辑。");
        }
        if (file.isMapper()) {
            reason.append("MyBatis Mapper，数据库访问层。");
        }
        if (file.containsEntity(task))) {
            reason.append("包含目标实体类型。");
        }

        return reason.length() > 0 ? reason.toString() : "通用代码文件";
    }

    private List<String> identifyGaps(SourceFile file, String task) {
        List<String> gaps = new ArrayList<>();

        // 检查缺失的依赖
        if (file.usesService() && !hasCorrespondingService(file)) {
            gaps.add("缺少对应的服务实现");
        }

        // 检查缺失的实体定义
        if (file.referencesEntity() && !hasEntityDefinition(file)) {
            gaps.add("缺少实体定义");
        }

        // 检查缺失的配置
        if (file.needsConfiguration() && !hasConfiguration(file)) {
            gaps.add("缺少 Spring 配置");
        }

        return gaps;
    }
}

/**
 * 评分标准：
 * - 高（0.8-1.0）：直接实现目标功能
 * - 中（0.5-0.7）：包含相关模式或类型
 * - 低（0.2-0.4）：间接相关
 * - 无（0-0.2）：不相关，排除
 */
```

### 阶段 3：REFINE

基于评估更新搜寻标准：

```java
package com.example.project.ai.context;

import java.util.Set;
import java.util.stream.Collectors;

/**
 * 查询精炼器
 */
public class QueryRefiner {

    public RetrievalQuery refine(
            List<RelevanceEvaluation> evaluation,
            RetrievalQuery previousQuery) {

        // 从高相关性文件中提取新模式
        Set<String> newPatterns = extractPatterns(evaluation);

        // 从代码库中找到新术语
        Set<String> newKeywords = extractKeywords(evaluation);

        // 排除确认不相关的路径
        Set<String> newExcludes = evaluation.stream()
                .filter(RelevanceEvaluation::isLowRelevance)
                .map(e -> e.file().path())
                .collect(Collectors.toSet());

        // 针对特定缺口
        Set<String> focusAreas = extractFocusAreas(evaluation);

        return new RetrievalQuery(
            mergePatterns(previousQuery.patterns(), newPatterns),
            mergeKeywords(previousQuery.keywords(), newKeywords),
            mergeExcludes(previousQuery.excludes(), newExcludes),
            focusAreas
        );
    }

    private Set<String> extractPatterns(List<RelevanceEvaluation> evaluation) {
        return evaluation.stream()
                .filter(RelevanceEvaluation::isHighRelevance)
                .flatMap(e -> e.file().relatedPatterns().stream())
                .collect(Collectors.toSet());
    }

    private Set<String> extractKeywords(List<RelevanceEvaluation> evaluation) {
        return evaluation.stream()
                .filter(RelevanceEvaluation::isHighRelevance)
                .flatMap(e -> e.file().extractKeywords().stream())
                .filter(kw -> !isCommonKeyword(kw))
                .collect(Collectors.toSet());
    }

    private Set<String> extractFocusAreas(List<RelevanceEvaluation> evaluation) {
        return evaluation.stream()
                .flatMap(e -> e.missingContext().stream())
                .distinct()
                .collect(Collectors.toSet());
    }

    private boolean isCommonKeyword(String keyword) {
        return Set.of("public", "private", "class", "return", "import")
                .contains(keyword);
    }
}
```

### 阶段 4：LOOP

以精炼标准重复（最多 3 个循环）：

```java
package com.example.project.ai.context;

import java.util.ArrayList;
import java.util.List;

/**
 * 迭代检索引擎
 */
public class IterativeRetrievalEngine {

    private final CodebaseRetriever retriever;
    private final RelevanceEvaluator evaluator;
    private final QueryRefiner refiner;
    private static final int MAX_CYCLES = 3;
    private static final double MIN_RELEVANCE = 0.7;
    private static final int MIN_HIGH_RELEVANCE_FILES = 3;

    /**
     * 执行迭代检索
     */
    public List<SourceFile> retrieve(String task) {
        RetrievalQuery query = RetrievalQuery.fromTask(task);
        List<SourceFile> bestContext = new ArrayList<>();

        for (int cycle = 0; cycle < MAX_CYCLES; cycle++) {
            // 1. 检索候选文件
            List<SourceFile> candidates = retriever.retrieve(query);

            // 2. 评估相关性
            List<RelevanceEvaluation> evaluation = evaluator.evaluate(candidates, task);

            // 3. 检查是否有足够上下文
            List<SourceFile> highRelevance = evaluation.stream()
                    .filter(RelevanceEvaluation::isHighRelevance)
                    .map(RelevanceEvaluation::file)
                    .toList();

            if (isSufficientContext(highRelevance, evaluation)) {
                return highRelevance;
            }

            // 4. 精炼查询并继续
            query = refiner.refine(evaluation, query);
            bestContext = mergeContext(bestContext, highRelevance);
        }

        return bestContext;
    }

    private boolean isSufficientContext(
            List<SourceFile> highRelevance,
            List<RelevanceEvaluation> evaluation) {

        if (highRelevance.size() >= MIN_HIGH_RELEVANCE_FILES) {
            return !hasCriticalGaps(evaluation);
        }

        return false;
    }

    private boolean hasCriticalGaps(List<RelevanceEvaluation> evaluation) {
        return evaluation.stream()
                .filter(RelevanceEvaluation::isHighRelevance)
                .anyMatch(e -> e.missingContext().contains("缺少实体定义"));
    }

    private List<SourceFile> mergeContext(
            List<SourceFile> existing,
            List<SourceFile> additional) {

        List<SourceFile> merged = new ArrayList<>(existing);
        additional.stream()
                .filter(file -> !existing.contains(file))
                .forEach(merged::add);
        return merged;
    }
}
```

## 实际示例

### 示例 1：Bug 修复上下文

```
任务："修复 JWT token 过期后无法刷新的问题"

循环 1：
  DISPATCH：在 src/main/java/** 搜寻 "token"、"jwt"、"refresh"
  EVALUATE：
    - JwtTokenProvider.java (0.95) - JWT 生成和验证
    - TokenController.java (0.85) - Token 端点
    - UserService.java (0.3) - 用户管理，间接相关
  REFINE：新增 "expiration"、"claims" 关键字；排除 UserService.java

循环 2：
  DISPATCH：搜寻精炼术语
  EVALUATE：
    - RefreshTokenService.java (0.98) - 刷新令牌服务
    - SecurityConfig.java (0.75) - Spring Security 配置
  REFINE：足够上下文（3 个高相关性文件）

结果：
  - JwtTokenProvider.java
  - TokenController.java
  - RefreshTokenService.java
  - SecurityConfig.java
```

### 示例 2：功能实现

```
任务："为用户管理 API 添加分页查询功能"

循环 1：
  DISPATCH：在 controller/** 搜寻 "user"、"page"、"query"
  EVALUATE：
    - UserController.java (0.9) - 用户控制器
    - UserService.java (0.8) - 用户服务
  REFINE：发现使用 MyBatis-Plus，新增 "Page"、"IPage" 关键字

循环 2：
  DISPATCH：搜寻 MyBatis-Plus 分页模式
  EVALUATE：
    - UserMapper.java (0.85) - Mapper 接口
    - MybatisPlusConfig.java (0.95) - 分页插件配置
  REFINE：需要查询包装器模式

循环 3：
  DISPATCH：搜寻 "LambdaQueryWrapper"、"QueryWrapper"
  EVALUATE：
    - QueryHelper.java (0.8) - 查询辅助工具
  REFINE：足够上下文

结果：
  - UserController.java
  - UserService.java
  - UserMapper.java
  - MybatisPlusConfig.java
  - QueryHelper.java
```

### 示例 3：事务管理问题

```
任务："修复订单创建时库存扣减的事务问题"

循环 1：
  DISPATCH：搜寻 "order"、"stock"、"transaction"
  EVALUATE：
    - OrderController.java (0.7) - 订单控制器
    - OrderService.java (0.85) - 订单服务
    - StockService.java (0.8) - 库存服务
  REFINE：新增 "@Transactional"、"propagation" 关键字

循环 2：
  DISPATCH：搜寻事务相关配置和模式
  EVALUATE：
    - TransactionConfig.java (0.9) - 事务配置
    - OrderServiceImpl.java (0.95) - 订单实现（含事务注解）
  REFINE：发现需要了解分布式事务模式

循环 3：
  DISPATCH：搜寻 "distributed"、"transaction"
  EVALUATE：
    - DistributedTransactionManager.java (0.75) - 分布式事务管理
  REFINE：足够上下文

结果：
  - OrderService.java
  - StockService.java
  - OrderServiceImpl.java
  - TransactionConfig.java
  - DistributedTransactionManager.java
```

## 与 Spring Boot 项目整合

### 源文件模型

```java
package com.example.project.ai.context;

import java.nio.file.Path;
import java.util.List;
import java.util.Set;

/**
 * 代表代码库中的一个源文件
 */
public record SourceFile(
        Path path,
        String content,
        Set<String> imports,
        Set<String> annotations,
        Set<String> referencedTypes
) {
    public boolean isController() {
        return annotations.contains("RestController") ||
               annotations.contains("Controller");
    }

    public boolean isService() {
        return annotations.contains("Service");
    }

    public boolean isMapper() {
        return annotations.contains("Mapper") ||
               path.toString().contains("mapper");
    }

    public boolean isEntity() {
        return annotations.contains("Entity") ||
               annotations.contains("Table");
    }

    public boolean isConfiguration() {
        return annotations.contains("Configuration");
    }

    public Set<String> relatedPatterns() {
        Set<String> patterns = new HashSet<>();

        if (isController()) {
            String pkg = packagePath();
            patterns.add(pkg + "/service/*.java");
            patterns.add(pkg + "/dto/**/*.java");
        }

        if (isService()) {
            String pkg = packagePath();
            patterns.add(pkg + "/mapper/*.java");
            patterns.add(pkg + "/entity/*.java");
        }

        return patterns;
    }

    public Set<String> extractKeywords() {
        Set<String> keywords = new HashSet<>();

        // 提取类名
        keywords.add(className());

        // 提取注解参数中的关键字
        for (String annotation : annotations) {
            if (annotation.contains("RequestMapping")) {
                keywords.add(extractPath(annotation));
            }
        }

        return keywords;
    }

    private String packagePath() {
        String pkg = extractPackage();
        return "/src/main/java/" + pkg.replace('.', '/');
    }

    private String className() {
        String filename = path.getFileName().toString();
        return filename.replace(".java", "");
    }

    private String extractPackage() {
        // 简化的包提取逻辑
        return content.lines()
                .filter(line -> line.startsWith("package "))
                .findFirst()
                .map(line -> line.replace("package ", "").replace(";", ""))
                .orElse("");
    }
}
```

### 代码库检索器

```java
package com.example.project.ai.context;

import org.springframework.stereotype.Component;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.stream.Stream;

/**
 * 代码库检索器
 */
@Component
public class CodebaseRetriever {

    private final Path projectRoot;

    public List<SourceFile> retrieve(RetrievalQuery query) {
        try {
            return Files.walk(projectRoot)
                    .filter(this::isJavaFile)
                    .filter(path -> matchesPattern(path, query.patterns()))
                    .filter(path -> !matchesExcludes(path, query.excludes()))
                    .map(this::loadSourceFile)
                    .filter(file -> matchesKeywords(file, query.keywords()))
                    .toList();
        } catch (IOException e) {
            throw new ContextRetrievalException("检索代码库失败", e);
        }
    }

    private boolean isJavaFile(Path path) {
        return path.toString().endsWith(".java");
    }

    private boolean matchesPattern(Path path, Set<String> patterns) {
        String pathStr = path.toString();
        return patterns.stream()
                .anyMatch(pattern -> pathStr.contains(pattern.replace("**/", "")));
    }

    private boolean matchesExcludes(Path path, Set<String> excludes) {
        return excludes.stream()
                .anyMatch(exclude -> path.toString().contains(exclude.replace("**/", "")));
    }

    private boolean matchesKeywords(SourceFile file, Set<String> keywords) {
        if (keywords.isEmpty()) {
            return true;
        }

        String content = file.content().toLowerCase();
        return keywords.stream()
                .anyMatch(kw -> content.contains(kw.toLowerCase()));
    }

    private SourceFile loadSourceFile(Path path) {
        try {
            String content = Files.readString(path);
            return new SourceFile(
                    path,
                    content,
                    extractImports(content),
                    extractAnnotations(content),
                    extractReferencedTypes(content)
            );
        } catch (IOException e) {
            throw new ContextRetrievalException("无法读取文件: " + path, e);
        }
    }

    private Set<String> extractImports(String content) {
        return content.lines()
                .filter(line -> line.startsWith("import "))
                .map(line -> line.replace("import ", "").replace(";", ""))
                .collect(Collectors.toSet());
    }

    private Set<String> extractAnnotations(String content) {
        Set<String> annotations = new HashSet<>();
        content.lines()
                .filter(line -> line.trim().startsWith("@"))
                .forEach(line -> {
                    String annotation = line.trim().split("[\\s\\(]")[0];
                    annotations.add(annotation.substring(1));
                });
        return annotations;
    }

    private Set<String> extractReferencedTypes(String content) {
        // 简化的类型提取逻辑
        return Set.of();
    }
}
```

### Spring Boot 集成服务

```java
package com.example.project.ai.service;

import com.example.project.ai.context.IterativeRetrievalEngine;
import com.example.project.ai.context.SourceFile;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.List;

/**
 * AI 上下文服务
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class AiContextService {

    private final IterativeRetrievalEngine retrievalEngine;

    /**
     * 为 AI 任务检索相关上下文
     */
    public List<SourceFile> retrieveContextForTask(String taskDescription) {
        log.info("开始检索任务上下文: {}", taskDescription);

        List<SourceFile> context = retrievalEngine.retrieve(taskDescription);

        log.info("检索完成，找到 {} 个相关文件", context.size());
        context.forEach(file ->
            log.debug("  - {}", file.path())
        );

        return context;
    }

    /**
     * 格式化上下文为 AI 提示
     */
    public String formatAsPrompt(List<SourceFile> context) {
        StringBuilder prompt = new StringBuilder();
        prompt.append("# 相关代码上下文\n\n");

        for (SourceFile file : context) {
            prompt.append("## ").append(file.path()).append("\n\n");
            prompt.append("```java\n");
            prompt.append(file.content());
            prompt.append("\n```\n\n");
        }

        return prompt.toString();
    }
}
```

## 在 Agent 提示中使用

```markdown
为 Java Spring Boot 项目检索上下文时，遵循迭代检索模式：

1. **DISPATCH**：从广泛关键字搜寻开始
   - 模式：**/*.java, **/*.xml, **/*.yml
   - 关键字：从任务描述中提取
   - 排除：**/test/**, **/*Test.java

2. **EVALUATE**：评估每个文件的相关性（0-1 尺度）
   - 高（>=0.7）：Controller/Service/Mapper 直接相关
   - 中（0.5-0.7）：Entity/DTO/Config 相关
   - 低（<0.2）：测试文件或不相关代码

3. **REFINE**：识别缺失的上下文
   - 检查依赖的服务/Mapper 是否包含
   - 检查实体定义是否存在
   - 检查 Spring 配置是否完整

4. **LOOP**：精炼搜寻标准并重复（最多 3 个循环）
   - 从高相关性文件中提取新模式
   - 学习项目特定术语（如使用 "Page" 而非 "paginate"）
   - 排除低相关性文件

5. **返回**：返回相关性 >= 0.7 的文件

## Spring Boot 特定模式识别：

- **Controller 层**：@RestController, @Controller, @RequestMapping
- **Service 层**：@Service, @Transactional
- **持久层**：@Mapper, extends BaseMapper
- **实体层**：@Entity, @Table, @Data
- **配置层**：@Configuration, @Bean, @ConfigurationProperties
```

## 最佳实践

1. **从广泛开始，逐渐缩小** - 不要过度指定初始查询
2. **学习代码库术语** - 第一个循环通常会揭示命名惯例
3. **追踪缺失内容** - 明确的缺口识别驱动精炼
4. **在"足够好"时停止** - 3 个高相关性文件胜过 10 个普通文件
5. **自信地排除** - 低相关性文件不会变得相关
6. **理解 Spring 分层** - Controller → Service → Mapper → Entity
7. **识别 MyBatis-Plus 模式** - LambdaQueryWrapper、IService、BaseMapper

## 相关

- [Longform Guide](https://x.com/affaanmustafa/status/2014040193557471352) - 子 agent 协调章节
- `continuous-learning` 技能 - 用于随时间改进的模式
- `springboot-patterns` 技能 - Spring Boot 特定模式
- `~/.claude/agents/` 中的 Agent 定义
