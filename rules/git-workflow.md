---
name: git-workflow
description: Git 工作流规范
priority: should
tags: [git, workflow, branch, commit, pr]
---

# Git 工作流

> **适用范围**：所有项目类型

## 分支命名规范

| 分支类型 | 命名格式 | 说明 | 示例 |
|----------|----------|------|------|
| **feature** | `feature/<description>` | 新功能开发 | `feature/user-auth` |
| **bugfix** | `bugfix/<description>` | Bug 修复 | `bugfix/login-crash` |
| **hotfix** | `hotfix/<description>` | 生产环境紧急修复 | `hotfix/security-patch` |
| **refactor** | `refactor/<description>` | 代码重构 | `refactor/user-service` |
| **docs** | `docs/<description>` | 文档更新 | `docs/api-readme` |

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
| **refactor** | 代码重构 | `refactor: 重构 UserService 依赖注入` |
| **docs** | 文档更新 | `docs: 更新 API 文档` |
| **test** | 测试相关 | `test: 添加用户注册单元测试` |
| **chore** | 构建/工具/依赖 | `chore: 升级 Spring Boot 到 3.2` |

### 署名配置

```json
// ~/.claude/settings.json
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
git checkout main
git merge --squash feature/user-auth
git commit -m "feat: 添加用户认证功能"
```

**优点**：主分支历史保持整洁
**缺点**：丢失开发过程中的 commit 历史

### Merge Commit（推荐用于团队协作）

```bash
git checkout main
git merge feature/user-auth
```

**优点**：保留完整的开发历史
**缺点**：历史记录较为复杂

### Rebase（谨慎使用）

```bash
# 仅对本地未推送的分支使用 rebase
git checkout feature/user-auth
git fetch origin
git rebase origin/main  # 安全：仅影响本地分支

# 如果需要强制推送
git push --force-with-lease origin feature/user-auth
```

**禁忌场景**：
- 绝对不要对已推送的公共分支执行 rebase
- 避免在多人协作的功能分支上使用 rebase

## Pull Request 工作流

### 创建 PR 前准备

```bash
# 1. 确保分支与主分支同步
git fetch origin
git rebase origin/main

# 2. 查看完整变更
git diff main...HEAD

# 3. 推送到远程
git push -u origin feature/user-auth
```

### PR 模板

```markdown
## 变更概述
- [ ] 新功能
- [ ] Bug 修复
- [ ] 性能优化
- [ ] 重构
- [ ] 文档更新

## 主要变更
1. **模块 A**: 添加用户登录功能
2. **模块 B**: 修复 token 验证逻辑
3. **测试**: 补充单元测试，覆盖率达到 85%

## 测试计划
- [ ] 本地单元测试通过
- [ ] 集成测试通过
- [ ] 代码审查通过

## 相关 Issue
Closes #123
```

## 代码审查流程

### 1. 编写代码后立即审查

**Java 项目：**
- 使用 **java-reviewer** Agent 进行主动审查

**其他项目：**
- 人工审查代码质量、安全性、性能

### 2. 处理审查反馈

| 问题级别 | 处理方式 | 示例 |
|----------|----------|------|
| **Critical** | 必须修复后合并 | SQL 注入、安全漏洞 |
| **High** | 强烈建议修复 | 性能问题、资源泄漏 |
| **Medium** | 建议修复 | 代码风格、命名 |
| **Low** | 可选 | 注释补充、格式调整 |

### 3. 审查通过后合并

```bash
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

```
需求确认 → 分支规划 → TDD 开发 → 代码审查 → 测试验证 → PR 创建 → 合并发布
```

### 步骤 1：先规划

对于复杂功能：
- 使用 **Plan** Agent 制定实现计划
- 识别依赖项和风险

```bash
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

**Java 项目：** 使用 **java-reviewer** Agent

### 步骤 4：测试验证

```bash
mvn clean test
mvn jacoco:report
```

### 步骤 5：提交与推送

```bash
git add src/main/java/UserService.java
git add src/test/java/UserServiceTest.java
git commit -m "feat: 添加用户认证功能

- 实现登录/注册接口
- 添加 JWT token 验证
- 补充单元测试，覆盖率 85%

Co-Authored-By: Claude (glm-4.7) <noreply@anthropic.com>"

git push -u origin feature/user-auth
```

### 步骤 6：创建 PR

```bash
gh pr create \
  --title "feat: 添加用户认证功能" \
  --body "$(cat <<'EOF'
## 变更概述
实现用户登录和注册功能，包括 JWT token 验证。

## 主要变更
1. **AuthService**: 添加登录/注册逻辑
2. **JwtTokenProvider**: 实现 JWT token 生成和验证
3. **AuthController**: 添加认证相关 API

## 测试计划
- [x] 本地单元测试通过
- [x] 集成测试通过

Closes #123
EOF
)"
```

## 紧急修复流程（Hotfix）

```bash
# 1. 从生产分支创建 hotfix
git checkout -b hotfix/security-patch

# 2. 快速修复和测试
# ... 编写代码 ...
# ... 运行测试 ...

# 3. 提交并推送
git commit -m "hotfix: 修复安全漏洞"
git push origin hotfix/security-patch

# 4. 创建 PR 并加急审查
gh pr create --label "hotfix,priority:critical"

# 5. 合并后同步回开发分支
git checkout main
git merge hotfix/security-patch
git push origin main
```

## 常见问题处理

### 合并冲突

```bash
# 方法 1：使用 Rebase（推荐用于本地功能分支）
git checkout feature/user-auth
git fetch origin
git rebase origin/main
# 解决冲突后
git add <resolved-files>
git rebase --continue
git push --force-with-lease origin feature/user-auth

# 方法 2：使用 Merge（更安全）
git checkout feature/user-auth
git fetch origin
git merge origin/main
# 解决冲突后
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
git stash
git stash list
git stash pop
```

## Git 配置建议

```bash
# 设置默认分支名
git config --global init.defaultBranch main

# 设置拉取策略（默认 rebase）
git config --global pull.rebase true

# 设置推送策略（默认当前分支）
git config --global push.default simple
```
