# Git 工作流

> **适用范围**：此工作流适用于所有项目类型

## 分支命名规范

| 分支类型 | 命名格式 | 说明 | 示例 |
|----------|----------|------|------|
| **feature** | `feature/<description>` | 新功能开发 | `feature/user-auth` |
| **bugfix** | `bugfix/<description>` | Bug 修复 | `bugfix/login-crash` |
| **hotfix** | `hotfix/<description>` | 生产环境紧急修复 | `hotfix/security-patch` |
| **refactor** | `refactor/<description>` | 代码重构 | `refactor/user-service` |
| **release** | `release/<version>` | 版本发布 | `release/v1.0.0` |
| **docs** | `docs/<description>` | 文档更新 | `docs/api-readme` |

**分支创建命令：**

```bash
# 创建功能分支
git checkout -b feature/user-auth

# 创建 bugfix 分支（基于 main）
git checkout main && git pull && git checkout -b bugfix/login-crash

# 创建 hotfix 分支（基于生产分支）
git checkout -b hotfix/security-patch
```

## Commit 消息格式

```
<type>: <description>

[optional body]

[optional footer]
```

### Commit 类型

| 类型 | 说明 | 示例 |
|------|------|------|
| **feat** | 新功能 | `feat: 添加用户登录功能` |
| **fix** | Bug 修复 | `fix: 修复登录时 token 过期问题` |
| **refactor** | 代码重构（不改变功能） | `refactor: 重构 UserService 依赖注入` |
| **docs** | 文档更新 | `docs: 更新 API 文档` |
| **test** | 测试相关 | `test: 添加用户注册单元测试` |
| **chore** | 构建/工具/依赖 | `chore: 升级 Spring Boot 到 3.2` |

> **其他类型**：`perf`（性能优化）、`ci`（CI/CD 配置）、`style`（代码风格）可根据需要使用。

**命名注意事项：**
- 使用小写类型
- 使用英文（或与团队约定一致的中文）
- 描述简洁明了（不超过 50 字符）
- 不要以句号结尾

**完整 Commit 示例：**

```bash
# 简单 commit
git commit -m "feat: 添加用户注册接口"

# 带 body 的 commit
git commit -m "fix: 修复登录验证失败问题

- 修正 JWT token 解析逻辑
- 添加过期时间检查
- 增加错误日志记录

Closes #123"
```

**署名配置：**

署名通过 `~/.claude/settings.json` 全局配置：

```json
{
  "gitCommit": {
    "coAuthor": {
      "name": "Claude (glm-4.7)",
      "email": "noreply@anthropic.com"
    }
  }
}
```

## 合并策略

### Squash Merge（推荐用于功能分支）

```bash
# 将多个 commit 压缩为一个，保持主分支整洁
git checkout main
git merge --squash feature/user-auth
git commit -m "feat: 添加用户认证功能"
```

**优点：**
- 主分支历史保持整洁
- 避免功能开发的中间 commit

**缺点：**
- 丢失开发过程中的 commit 历史

### Merge Commit（推荐用于团队协作）

```bash
# 保留完整的分支历史
git checkout main
git merge feature/user-auth
```

**优点：**
- 保留完整的开发历史
- 清晰展示分支结构

**缺点：**
- 历史记录较为复杂

### Rebase（谨慎使用）

```bash
# 将当前分支的 commit 移到目标分支之上
git checkout feature/user-auth
git rebase main
```

**推荐场景：**
- 保持本地功能分支与主分支同步
- 清理本地 commit 历史（合并、排序、修改提交信息）
- 在创建 PR 前整理提交历史

**禁忌场景：**
- **绝对不要对已推送的公共分支执行 rebase**
- 避免在多人协作的功能分支上使用 rebase
- 不要在包含重要历史记录的分支上使用

**安全做法：**
```bash
# 仅对本地未推送的分支使用 rebase
git checkout feature/user-auth
git fetch origin
git rebase origin/main  # 安全：仅影响本地分支

# 如果需要强制推送，使用 --force-with-lease
git push --force-with-lease origin feature/user-auth
```

## Pull Request 工作流

### 创建 PR 前准备

```bash
# 1. 确保分支与主分支同步
git fetch origin
git rebase origin/main  # 仅本地分支，安全使用

# 2. 查看完整变更（从分叉点开始）
git diff main...HEAD

# 3. 查看分支独有的 commit
git log main..HEAD --oneline

# 4. 推送到远程（首次使用 -u）
git push -u origin feature/user-auth
```

### PR 模板

```markdown
## 变更概述
<!-- 简要描述此 PR 的目的 -->

- [ ] 新功能
- [ ] Bug 修复
- [ ] 性能优化
- [ ] 重构
- [ ] 文档更新
- [ ] 测试补充

## 主要变更
<!-- 列出主要的代码变更 -->

1. **模块 A**: 添加用户登录功能
2. **模块 B**: 修复 token 验证逻辑
3. **测试**: 补充单元测试，覆盖率达到 85%

## 测试计划
- [ ] 本地单元测试通过
- [ ] 集成测试通过
- [ ] 手动测试关键流程
- [ ] 代码审查通过

## 相关 Issue
Closes #123

## 截图/演示
<!-- 如果有 UI 变更，添加截图或 GIF -->

## Checklist
- [ ] 代码符合项目风格指南
- [ ] 已添加/更新测试
- [ ] 测试覆盖率 >= 80%
- [ ] 文档已更新
- [ ] 无安全漏洞
- [ ] 无性能回归
```

### PR 审查要点

**代码质量：**
- 代码逻辑清晰、易读
- 遵循项目编码规范
- 没有硬编码配置
- 适当的错误处理

**安全性：**
- 无 SQL 注入风险
- 无 XSS 漏洞
- 敏感数据已脱敏
- 依赖无已知漏洞

**测试：**
- 单元测试覆盖核心逻辑
- 集成测试验证关键流程
- 测试命名清晰、描述准确

**文档：**
- API 文档已更新
- README/CHANGELOG 已更新
- 复杂逻辑有注释说明

## 代码审查流程

### 1. 编写代码后立即审查

**Java 项目：**
```bash
# 使用 java-reviewer Agent 进行主动审查
```

**其他项目：**
```bash
# 人工审查清单
- [ ] 代码可读性
- [ ] 命名规范
- [ ] 错误处理
- [ ] 性能影响
- [ ] 安全漏洞
```

### 2. 处理审查反馈

| 问题级别 | 处理方式 | 示例 |
|----------|----------|------|
| **Critical** | 必须修复后合并 | SQL 注入、安全漏洞 |
| **High** | 强烈建议修复 | 性能问题、资源泄漏 |
| **Medium** | 建议修复 | 代码风格、命名 |
| **Low** | 可选 | 注释补充、格式调整 |

### 3. 审查通过后合并

```bash
# 方法 1：通过 GitHub/GitLab UI 合并

# 方法 2：命令行合并（Squash Merge）
git checkout main
git pull
git merge --squash feature/user-auth
git commit -m "feat: 添加用户认证功能"
git push origin main

# 删除已合并的分支
git branch -d feature/user-auth
git push origin --delete feature/user-auth
```

## 功能实现工作流

### 完整流程图

```
需求确认 → 分支规划 → TDD 开发 → 代码审查 → 测试验证 → PR 创建 → 合并发布
   ↓          ↓          ↓          ↓          ↓          ↓          ↓
Plan Agent  git       java-test  java-review e2e Agent   gh pr    git merge
           checkout
```

### 步骤 1：先规划

对于复杂功能：
- 使用 **Plan** Agent 制定实现计划
- 识别依赖项和风险
- 拆分为多个阶段

```bash
# 创建功能分支
git checkout -b feature/user-auth
```

### 步骤 2：TDD 方法（Java 项目）

```
RED   → 编写失败的测试
GREEN → 编写最小实现使测试通过
REFACTOR → 重构优化代码
VERIFY → 验证 80%+ 覆盖率
```

使用 **java-test** Agent 强制执行此流程。

### 步骤 3：代码审查

**Java 项目：**
- 编写代码后立即使用 **java-reviewer** Agent
- 处理关键和高优先级问题

**其他项目：**
- 人工审查代码质量
- 检查安全漏洞
- 验证错误处理

### 步骤 4：测试验证

```bash
# 运行所有测试
mvn clean test

# 检查覆盖率
mvn jacoco:report

# 运行集成测试（如有）
mvn verify
```

对于需要集成测试的场景，使用 **e2e** Agent 生成和运行集成测试（JUnit 5、RestAssured、TestContainers）。

### 步骤 5：提交与推送

```bash
# 查看变更
git status
git diff

# 暂存文件
git add src/main/java/UserService.java
git add src/test/java/UserServiceTest.java

# 提交（格式：type: description）
git commit -m "feat: 添加用户认证功能

- 实现登录/注册接口
- 添加 JWT token 验证
- 补充单元测试，覆盖率 85%

Co-Authored-By: Claude (glm-4.7) <noreply@anthropic.com>"

# 推送（首次使用 -u）
git push -u origin feature/user-auth
```

### 步骤 6：创建 PR

```bash
# 使用 gh CLI 创建 PR
gh pr create \
  --title "feat: 添加用户认证功能" \
  --body "$(cat <<'EOF'
## 变更概述
实现用户登录和注册功能，包括 JWT token 验证。

## 主要变更
1. **AuthService**: 添加登录/注册逻辑
2. **JwtTokenProvider**: 实现 JWT token 生成和验证
3. **AuthController**: 添加认证相关 API
4. **测试**: 补充单元测试，覆盖率 85%

## 测试计划
- [x] 本地单元测试通过
- [x] 集成测试通过
- [x] 代码审查通过

## 相关 Issue
Closes #123

---
🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

### 步骤 7：合并后清理

```bash
# 切换到主分支
git checkout main
git pull

# 删除本地分支
git branch -d feature/user-auth

# 删除远程分支
git push origin --delete feature/user-auth
```

## 紧急修复流程（Hotfix）

```bash
# 1. 从生产分支创建 hotfix
git checkout -b hotfix/security-patch

# 2. 快速修复和测试
# ... 编写代码 ...
# ... 运行测试 ...

# 3. 提交并推送到远程
git commit -m "hotfix: 修复安全漏洞"
git push origin hotfix/security-patch

# 4. 创建 PR 并加急审查
gh pr create --label "hotfix,priority:critical"

# 5. 合并后同步回开发分支
git checkout main
git merge hotfix/security-patch
git push origin main
```

## Git 配置建议

```bash
# 设置默认分支名
git config --global init.defaultBranch main

# 设置拉取策略（默认 rebase）
git config --global pull.rebase true

# 设置推送策略（默认当前分支）
git config --global push.default simple

# 设置别名（可选）
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.lg "log --graph --oneline --all"
```

## 常见问题处理

### 合并冲突

```bash
# 方法 1：使用 Rebase（推荐用于本地功能分支）
git checkout feature/user-auth
git fetch origin
git rebase origin/main
# 解决冲突后：
git add <resolved-files>
git rebase --continue
git push --force-with-lease origin feature/user-auth

# 方法 2：使用 Merge（更安全，保留历史）
git checkout feature/user-auth
git fetch origin
git merge origin/main
# 解决冲突后：
git add <resolved-files>
git commit -m "Merge conflict resolution"
git push origin feature/user-auth
```

### 撤销提交

```bash
# 撤销最后一次 commit（保留更改）
git reset --soft HEAD~1

# 撤销最后一次 commit（丢弃更改）
git reset --hard HEAD~1

# 撤销已推送的 commit（创建新 commit）
git revert <commit-hash>
```

### 暂存未完成工作

```bash
# 暂存当前更改
git stash

# 查看暂存列表
git stash list

# 恢复暂存的更改
git stash pop

# 恢复并删除暂存
git stash apply
```
