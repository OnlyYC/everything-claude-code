---
name: iterative-retrieval
description: 渐进精炼上下文检索模式，解决子 agent 上下文问题。通过 4 阶段循环（DISPATCH → EVALUATE → REFINE → LOOP）逐步收集相关代码文件。
version: 1.1.0
tech_stack: [Java 21, Spring Boot 3, MyBatis-Plus]
tools: Read, Grep, Glob
related_skills: [springboot-patterns, continuous-learning, strategic-compact]
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

## 4 阶段流程

### 1. DISPATCH - 初始广泛查询

从任务描述中提取关键字，进行广泛搜索：

```
patterns:  ["**/*.java", "**/*.xml", "**/*.yml"]
keywords:  从任务描述提取（分词、转小写）
excludes:  ["**/test/**", "**/*Test.java"]
focus:    []
```

### 2. EVALUATE - 评估相关性

对每个文件评分（0.0 - 1.0）：

| 分数范围 | 相关性 | 处理方式 |
|----------|--------|----------|
| 0.8 - 1.0 | 高 | 保留，提取模式 |
| 0.5 - 0.7 | 中 | 保留，可能需要补充 |
| 0.2 - 0.4 | 低 | 观察后决定 |
| 0 - 0.2 | 无 | 排除 |

**Spring Boot 特定评分：**
- `@RestController` / `@Controller`：+0.3
- `@Service`：+0.3
- `@Mapper` / extends `BaseMapper`：+0.3
- 包含目标实体类型：+0.2
- 关键字直接匹配：+0.4

### 3. REFINE - 精炼搜索标准

从高相关性文件中提取新信息：

```
newPatterns:  高相关性文件的依赖包路径
newKeywords:  高相关性文件的类名、方法名（排除常见词）
newExcludes:  低相关性文件的路径
focusAreas:  从缺失上下文识别（"缺少服务"、"缺少实体"等）
```

**常见关键词过滤：**
```
排除: public, private, class, return, import, void, String
保留: Order, UserService, createOrder, processPayment
```

### 4. LOOP - 重复最多 3 次

```
循环 1: 广泛关键字 → 初步筛选
循环 2: 学习项目术语 → 精确筛选
循环 3: 填补缺失上下文 → 最终结果

终止条件:
- 获得 3+ 个高相关性文件
- 无关键缺失上下文
- 达到最大循环次数 (3)
```

## Spring Boot 项目模式识别

### 分层架构识别

| 文件类型 | 识别特征 | 相关包路径 | 依赖方向 |
|----------|----------|-----------|---------|
| Controller | `@RestController`, `@Controller`, `@RequestMapping` | `controller/` → `service/` | 下 |
| Service | `@Service`, `@Transactional` | `service/` → `mapper/`, `entity/` | 下 |
| Mapper | `@Mapper`, extends `BaseMapper<T>` | `mapper/` → `entity/` | 下 |
| Entity | `@TableName`, `@TableId`, `@Data` | `entity/` | 无 |
| DTO | `@Data`, 命名含 `DTO`, `Request`, `Response` | `dto/`, `vo/` | 无 |
| Config | `@Configuration`, `@Bean` | `config/` | 可选 |
| Exception | extends `RuntimeException`, `@ControllerAdvice` | `exception/`, `handler/` | 无 |

### MyBatis-Plus 特定模式

| 模式 | 识别特征 | 相关文件 |
|------|----------|---------|
| IService | 接口 extends `IService<T>` | Service 接口 |
| ServiceImpl | 类 extends `ServiceImpl<M, T>` | Service 实现 |
| BaseMapper | 接口 extends `BaseMapper<T>` | Mapper 接口 |
| LambdaQueryWrapper | `LambdaQueryWrapper<>`, `.eq()`, `.like()` | Service/Controller |
| QueryWrapper | `QueryWrapper<>`, 字符串条件 | Service/Controller |
| Page<T> | `Page<>`, `IPage<>` | 分页查询 |

### Spring Security 模式

| 模式 | 识别特征 | 相关文件 |
|------|----------|---------|
| SecurityConfig | `@EnableWebSecurity`, `SecurityFilterChain` | `config/SecurityConfig.java` |
| JWT | `JwtTokenProvider`, `JwtAuthenticationFilter` | `security/` 或 `config/` |
| UserDetails | implements `UserDetailsService` | `security/` 或 `service/` |
| Permission | `@PreAuthorize`, `@Secured` | Controller/Service |

### 缓存模式

| 模式 | 识别特征 | 相关文件 |
|------|----------|---------|
| Redis Config | `RedisTemplate`, `RedisConnectionFactory` | `config/RedisConfig.java` |
| Cacheable | `@Cacheable`, `@CacheEvict` | Service 方法 |
| CacheManager | `RedisCacheManager`, `CacheManager` | `config/` |

### 异步处理模式

| 模式 | 识别特征 | 相关文件 |
|------|----------|---------|
| Async | `@Async`, `@EnableAsync` | Service, Config |
| Event | `@EventListener`, `ApplicationEvent` | Event, Listener |
| Message Queue | `@RabbitListener`, `@KafkaListener` | Listener, Config |

### 数据验证模式

| 模式 | 识别特征 | 相关文件 |
|------|----------|---------|
| Validation | `@Valid`, `@Validated`, `@NotNull` | Controller, DTO |
| Validator | implements `Validator` | `validator/` |
| ExceptionHandler | `@ExceptionHandler`, `@ControllerAdvice` | `exception/` |

## 实际示例

### 示例 1：Bug 修复上下文

```
任务："修复 JWT token 过期后无法刷新的问题"

循环 1:
  DISPATCH: 搜寻 "token", "jwt", "refresh"
  EVALUATE:
    - JwtTokenProvider.java (0.95) - JWT 生成和验证
    - TokenController.java (0.85) - Token 端点
    - UserService.java (0.3) - 用户管理，间接相关
  REFINE: 新增 "expiration", "claims" 关键字

循环 2:
  EVALUATE:
    - RefreshTokenService.java (0.98) - 刷新令牌服务
    - SecurityConfig.java (0.75) - Spring Security 配置
  REFINE: 足够上下文

结果: JwtTokenProvider.java, TokenController.java, RefreshTokenService.java, SecurityConfig.java
```

### 示例 2：功能实现

```
任务："为用户管理 API 添加分页查询功能"

循环 1:
  DISPATCH: 搜寻 "user", "page", "query"
  EVALUATE:
    - UserController.java (0.9)
    - UserService.java (0.8)
  REFINE: 发现 MyBatis-Plus，新增 "Page", "IPage" 关键字

循环 2:
  DISPATCH: 搜寻 MyBatis-Plus 分页模式
  EVALUATE:
    - UserMapper.java (0.85)
    - MybatisPlusConfig.java (0.95)
  REFINE: 需要查询包装器模式

循环 3:
  DISPATCH: 搜寻 "LambdaQueryWrapper", "QueryWrapper"
  EVALUATE:
    - QueryHelper.java (0.8)
  REFINE: 足够上下文
```

### 示例 3：性能优化

```
任务："优化订单查询性能，添加缓存"

循环 1:
  DISPATCH: 搜寻 "order", "query", "find"
  EVALUATE:
    - OrderController.java (0.85)
    - OrderService.java (0.9)
    - OrderMapper.java (0.75)
  REFINE: 需要缓存相关文件

循环 2:
  DISPATCH: 搜寻 "cache", "@Cacheable", "Redis"
  EVALUATE:
    - CacheConfig.java (0.95)
    - RedisConfig.java (0.9)
  REFINE: 需要序列化配置

循环 3:
  DISPATCH: 搜寻 "serializer", "RedisTemplate"
  EVALUATE:
    - RedisConfig.java (已获取)
    - CacheService.java (0.85)
  REFINE: 足够上下文
```

### 示例 4：重构任务

```
任务："重构支付服务，提取接口"

循环 1:
  DISPATCH: 搜寻 "payment", "pay", "service"
  EVALUATE:
    - PaymentService.java (0.95)
    - PaymentController.java (0.8)
  REFINE: 需要查看现有接口定义

循环 2:
  DISPATCH: 搜寻 "interface", "Payment"
  EVALUATE:
    - IPaymentService.java (0.9) - 发现已存在接口
    - AlipayService.java (0.85) - 实现类
    - WechatPayService.java (0.85) - 实现类
  REFINE: 需要查看工厂模式

循环 3:
  DISPATCH: 搜寻 "PaymentFactory", "Strategy"
  EVALUATE:
    - PaymentFactory.java (0.9)
    - PaymentStrategy.java (0.88)
  REFINE: 足够上下文
```

### 示例 5：安全审计

```
任务："审计 SQL 注入风险"

循环 1:
  DISPATCH: 搜寻 "*", "mapper", "query"
  EVALUATE:
    - UserMapper.java (0.7)
    - OrderMapper.java (0.7)
    - ProductMapper.java (0.7)
  REFINE: 需要查看 XML 文件

循环 2:
  DISPATCH: 搜寻 "*.xml", "select", "${"
  EVALUATE:
    - UserMapper.xml (0.9) - 包含 ${param} 使用
    - OrderMapper.xml (0.85) - 使用 #{} 安全
    - ReportMapper.xml (0.95) - 多处 ${}
  REFINE: 需要检查参数化查询

循环 3:
  DISPATCH: 搜寻 "QueryWrapper", "LambdaQueryWrapper"
  EVALUATE:
    - BaseQuery.java (0.8) - 包装器基类
  REFINE: 足够上下文
```

### 示例 6：测试编写

```
任务："为 MarketService 编写单元测试"

循环 1:
  DISPATCH: 搜寻 "MarketService"
  EVALUATE:
    - MarketService.java (0.95)
  REFINE: 需要依赖和测试模式

循环 2:
  DISPATCH: 搜寻 "MarketMapper", "MarketEntity"
  EVALUATE:
    - MarketMapper.java (0.9)
    - MarketEntity.java (0.85)
  REFINE: 需要查看现有测试模式

循环 3:
  DISPATCH: 搜寻 "*Test.java", "Mockito"
  EVALUATE:
    - UserServiceTest.java (0.9) - 测试模式参考
    - OrderServiceTest.java (0.88)
  REFINE: 足够上下文
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

- `springboot-patterns` - Spring Boot 特定模式
- `continuous-learning` - 持续学习技能
- `~/.claude/agents/` 中的 Agent 定义

## 实现参考

### Java 版本伪代码

```java
/**
 * 迭代检索算法伪代码 (Java 21)
 *
 * 核心思想：通过多轮迭代，逐步精炼搜索条件，找到最相关的文件
 *
 * @see IterativeRetrieval#iterativeRetrieval(String) 主入口方法
 */
public class IterativeRetrieval {

    private static final int MAX_LOOPS = 3;                      // 最大迭代次数
    private static final double HIGH_RELEVANCE_THRESHOLD = 0.7; // 高相关性阈值
    private static final int MIN_HIGH_RELEVANCE_FILES = 3;      // 最少高相关文件数

    /**
     * 执行迭代检索
     *
     * @param taskDescription 任务描述，用于提取关键字
     * @return 高相关性文件路径列表
     */
    public List<Path> iterativeRetrieval(String taskDescription) {
        // 初始搜索条件
        List<String> patterns = List.of("**/*.java", "**/*.xml", "**/*.yml");
        Set<String> keywords = extractKeywords(taskDescription);
        Set<String> excludes = Set.of("**/test/**", "**/*Test.java");
        Set<String> focusAreas = new HashSet<>();

        List<ScoredFile> highRelevanceFiles = new ArrayList<>();

        // 迭代搜索
        for (int loop = 0; loop < MAX_LOOPS; loop++) {
            // 步骤 1: DISPATCH - 搜索文件
            List<Path> files = searchFiles(patterns, keywords, excludes);

            // 步骤 2: EVALUATE - 评分
            List<ScoredFile> scoredFiles = scoreRelevance(files, taskDescription);
            highRelevanceFiles = scoredFiles.stream()
                .filter(f -> f.score() >= HIGH_RELEVANCE_THRESHOLD)
                .toList();

            // 检查终止条件
            if (highRelevanceFiles.size() >= MIN_HIGH_RELEVANCE_FILES) {
                break;
            }

            // 步骤 3: REFINE - 提取新模式
            Set<String> newPatterns = extractPackagePaths(highRelevanceFiles);
            Set<String> newKeywords = extractClassNames(highRelevanceFiles);

            keywords.addAll(newKeywords);
            patterns = Stream.concat(patterns.stream(), newPatterns.stream())
                .toList();
        }

        return highRelevanceFiles.stream()
            .map(ScoredFile::path)
            .toList();
    }

    /**
     * 计算文件相关性分数
     *
     * @param file 文件路径
     * @param taskDescription 任务描述
     * @return 相关性分数 (0.0 - 1.0+)
     */
    private double scoreRelevance(Path file, String taskDescription) {
        double score = 0.0;
        String content = readFile(file);

        // Spring Boot 特定评分
        if (content.contains("@RestController")) score += 0.3;
        if (content.contains("@Service")) score += 0.3;
        if (content.contains("@Mapper") || content.contains("BaseMapper")) score += 0.3;

        // 关键字匹配
        for (String keyword : extractKeywords(taskDescription)) {
            if (content.contains(keyword)) score += 0.4;
        }

        return score;
    }

    /**
     * 从文件中提取包路径
     */
    private Set<String> extractPackagePaths(List<ScoredFile> files) {
        return files.stream()
            .flatMap(f -> extractPackageFromPath(f.path()).stream())
            .collect(Collectors.toSet());
    }

    /**
     * 从文件中提取类名（排除常见关键字）
     */
    private Set<String> extractClassNames(List<ScoredFile> files) {
        Set<String> commonWords = Set.of("public", "private", "class", "return",
                                          "import", "void", "String");
        return files.stream()
            .flatMap(f -> extractClassNamesFromFile(f.path()).stream())
            .filter(name -> !commonWords.contains(name))
            .collect(Collectors.toSet());
    }
}

// 记录文件及其相关性分数
record ScoredFile(Path path, double score) {}
```

### Python 版本参考

```python
# 迭代检索算法伪代码
def iterative_retrieval(task_description, max_loops=3):
    patterns = ["**/*.java", "**/*.xml", "**/*.yml"]
    keywords = extract_keywords(task_description)
    excludes = ["**/test/**", "**/*Test.java"]
    focus_areas = []

    for loop in range(max_loops):
        # 1. DISPATCH - 搜索文件
        files = search_files(patterns, keywords, excludes)

        # 2. EVALUATE - 评分
        scored_files = score_relevance(files, task_description)
        high_relevance = [f for f in scored_files if f.score >= 0.7]

        if len(high_relevance) >= 3:
            break

        # 3. REFINE - 提取新模式
        new_patterns = extract_package_paths(high_relevance)
        new_keywords = extract_class_names(high_relevance)
        keywords.extend(new_keywords)
        patterns.extend(new_patterns)

    return [f.path for f in high_relevance]

def score_relevance(file, task_description):
    score = 0.0
    content = read_file(file)

    # Spring Boot 特定评分
    if "@RestController" in content: score += 0.3
    if "@Service" in content: score += 0.3
    if "@Mapper" in content or "BaseMapper" in content: score += 0.3

    # 关键字匹配
    for keyword in task_description.keywords:
        if keyword in content: score += 0.4

    return score
```

---

## 性能优化建议

### 搜索优化

```python
# ✅ 使用 glob 模式过滤，避免读取所有文件
patterns = [
    "**/controller/**/*.java",    # 只搜索 Controller 层
    "**/service/**/*.java",       # 只搜索 Service 层
    "**/mapper/**/*.java"         # 只搜索 Mapper 层
]

# ❌ 避免过于宽泛的搜索
patterns = ["**/*.java"]  # 会扫描整个项目

# ✅ 优先级排序
def prioritize_files(files, task_type):
    """根据任务类型排序文件"""
    priorities = {
        'controller': ['controller', 'rest'],
        'service': ['service', 'business'],
        'mapper': ['mapper', 'dao', 'repository'],
        'entity': ['entity', 'model', 'domain'],
        'config': ['config', 'configuration']
    }

    task_keywords = priorities.get(task_type, [])
    return sorted(files, key=lambda f: file_priority_score(f, task_keywords), reverse=True)
```

### 缓存策略

```python
# ✅ 文件内容缓存
from functools import lru_cache

@lru_cache(maxsize=256)
def read_file_cached(file_path: str) -> str:
    """缓存文件读取结果"""
    with open(file_path, 'r', encoding='utf-8') as f:
        return f.read()

# ✅ 搜索结果缓存
class RetrievalCache:
    def __init__(self, max_size=100):
        self.cache = {}
        self.max_size = max_size

    def get(self, key):
        return self.cache.get(key)

    def set(self, key, value):
        if len(self.cache) >= self.max_size:
            # 移除最旧的条目
            oldest = next(iter(self.cache))
            del self.cache[oldest]
        self.cache[key] = value

# 使用方式
cache = RetrievalCache()

def search_with_cache(query: str):
    cached = cache.get(query)
    if cached:
        return cached

    result = perform_search(query)
    cache.set(query, result)
    return result
```

### 并行处理

```python
# ✅ 并行文件读取
from concurrent.futures import ThreadPoolExecutor, as_completed

def read_files_parallel(file_paths: list[str]) -> dict[str, str]:
    """并行读取多个文件"""
    results = {}

    with ThreadPoolExecutor(max_workers=8) as executor:
        futures = {
            executor.submit(read_file, path): path
            for path in file_paths
        }

        for future in as_completed(futures):
            path = futures[future]
            try:
                results[path] = future.result()
            except Exception as e:
                print(f"Error reading {path}: {e}")

    return results

# ✅ 并行搜索
def search_patterns_parallel(patterns: list[str]) -> list[str]:
    """并行搜索多个模式"""
    from glob import glob

    with ThreadPoolExecutor() as executor:
        futures = [executor.submit(glob, p) for p in patterns]
        results = []
        for future in as_completed(futures):
            results.extend(future.result())

    return results
```

### 内存优化

```python
# ✅ 流式处理大文件
def read_file_streaming(file_path: str, max_lines: int = 1000):
    """流式读取文件，避免一次性加载"""
    relevant_lines = []
    with open(file_path, 'r', encoding='utf-8') as f:
        for i, line in enumerate(f):
            if i >= max_lines:
                break
            relevant_lines.append(line)
    return ''.join(relevant_lines)

# ✅ 惰性求值
def iter_relevant_files(base_path: str, patterns: list[str]):
    """惰性迭代文件，不一次性加载所有"""
    for pattern in patterns:
        for file_path in Path(base_path).rglob(pattern[2:]):  # 移除 **
            yield file_path

# 使用方式
for file in iter_relevant_files(src_dir, patterns):
    if is_relevant(file):
        process_file(file)
```

### 提前终止

```python
# ✅ 满意度终止
def search_until_satisfied(query: str, min_files: int = 3, min_score: float = 0.8):
    """找到足够的高质量文件后停止"""
    high_quality_files = []

    for file in iter_files():
        score = calculate_relevance(file, query)

        if score >= min_score:
            high_quality_files.append(file)

            if len(high_quality_files) >= min_files:
                break  # 提前终止

    return high_quality_files

# ✅ 收益递减检测
def has_diminishing_returns(current_scores: list[float]) -> bool:
    """检测是否继续搜索还有价值"""
    if len(current_scores) < 5:
        return False

    # 检查最近 5 次搜索的平均分是否在下降
    recent_avg = sum(current_scores[-5:]) / 5
    overall_avg = sum(current_scores) / len(current_scores)

    return recent_avg < overall_avg * 0.7
```

---

## 工具集成

### 与 Glob 工具集成

```python
# ✅ 使用 Glob 工具进行模式搜索
import glob

def find_files_by_pattern(base_dir: str, pattern: str) -> list[str]:
    """使用 Glob 查找文件"""
    search_pattern = f"{base_dir}/{pattern}"
    return glob.glob(search_pattern, recursive=True)

# 示例
files = find_files_by_pattern("src/main/java", "**/controller/**/*.java")
```

### 与 Grep 工具集成

```python
# ✅ 使用 Grep 工具进行内容搜索
import subprocess

def grep_content(pattern: str, files: list[str]) -> dict[str, list[str]]:
    """在文件中搜索模式"""
    results = {}

    for file in files:
        try:
            output = subprocess.check_output(
                ["grep", "-n", pattern, file],
                text=True,
                stderr=subprocess.DEVNULL
            )
            if output.strip():
                results[file] = output.strip().split('\n')
        except subprocess.CalledProcessError:
            continue

    return results

# 示例
matches = grep_content("@RestController", java_files)
```

---

## 高级技巧

### 语义相似度

```python
# ✅ 使用词干提取和词形还原
from nltk.stem import PorterStemmer

stemmer = PorterStemmer()

def normalize_keywords(keywords: list[str]) -> set[str]:
    """规范化关键词"""
    normalized = set()
    for keyword in keywords:
        # 转小写
        keyword = keyword.lower()
        # 提取词干
        stemmed = stemmer.stem(keyword)
        normalized.add(stemmed)
    return normalized

# ✅ TF-IDF 相似度
from sklearn.feature_extraction.text import TfidfVectorizer

def calculate_similarity(query: str, file_content: str) -> float:
    """计算查询与文件内容的相似度"""
    vectorizer = TfidfVectorizer()
    tfidf = vectorizer.fit_transform([query, file_content])

    # 余弦相似度
    similarity = (tfidf * tfidf.T).A[0, 1]
    return similarity
```

### 上下文推断

```python
# ✅ 从导入语句推断依赖
import re

def extract_dependencies(java_file: str) -> list[str]:
    """从 Java 文件提取导入的依赖"""
    with open(java_file, 'r') as f:
        content = f.read()

    imports = re.findall(r'import\s+([\w.]+);', content)

    # 提取相关的包路径
    dependencies = []
    for imp in imports:
        # 转换包路径为文件路径
        if '.entity.' in imp:
            entity = imp.split('.')[-1]
            dependencies.append(f"**/{entity}.java")
        elif '.service.' in imp:
            service = imp.split('.')[-1]
            dependencies.append(f"**/service/**/{service}.java")

    return dependencies

# ✅ 从注解推断职责
def infer_responsibility(java_file: str) -> str:
    """从注解推断文件的职责"""
    with open(java_file, 'r') as f:
        content = f.read()

    if '@RestController' in content or '@Controller' in content:
        return 'controller'
    elif '@Service' in content:
        return 'service'
    elif '@Mapper' in content or 'extends BaseMapper' in content:
        return 'mapper'
    elif '@Entity' in content or '@TableName' in content:
        return 'entity'
    elif '@Configuration' in content:
        return 'config'
    else:
        return 'unknown'
```

---

## 故障排除

### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 找不到相关文件 | 关键词不匹配 | 检查任务描述，提取更多同义词 |
| 找到太多文件 | 搜索条件太宽泛 | 增加排除模式，提高评分阈值 |
| 评分不准确 | 评分规则不适应项目 | 调整权重，添加项目特定规则 |
| 循环次数过多 | 终止条件不合理 | 检查高相关性文件数量 |
| 性能问题 | 读取大量文件 | 使用缓存、并行处理 |

### 调试技巧

```python
# ✅ 记录搜索过程
def debug_search(task: str):
    """调试搜索过程"""
    print(f"\n=== 搜索任务: {task} ===")

    keywords = extract_keywords(task)
    print(f"提取的关键词: {keywords}")

    files = search_files(keywords)
    print(f"找到 {len(files)} 个文件")

    for file in files:
        score = calculate_score(file, task)
        print(f"  {file}: {score:.2f}")

    print(f"\n最终选择: {selected_files}")

# ✅ 可视化相关性
import matplotlib.pyplot as plt

def visualize_relevance(scores: dict[str, float]):
    """可视化文件相关性分数"""
    files = list(scores.keys())
    values = list(scores.values())

    plt.figure(figsize=(12, 6))
    plt.barh(files, values)
    plt.xlabel('Relevance Score')
    plt.title('File Relevance')
    plt.tight_layout()
    plt.show()
```

---

## 相关技能

- `springboot-patterns` - Spring Boot 特定模式
- `continuous-learning` - 持续学习技能
- `strategic-compact` - 策略性上下文压缩
- `verification-loop` - 项目验证流程
