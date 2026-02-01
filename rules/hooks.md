# Hook 系统

## Hook 类型

- **PreToolUse**：工具执行前（验证、参数修改）
- **PostToolUse**：工具执行后（自动格式化、检查）
- **Stop**：会话结束时（最终验证）

## 当前 Hooks（在 ~/.claude/settings.json）

### PreToolUse
- **tmux 提醒**：建议对长时间运行的命令使用 tmux（npm、pnpm、yarn、cargo 等）
- **git push 审查**：推送前开启 Zed 进行审查
- **文档阻止器**：阻止创建不必要的 .md/.txt 文件

### PostToolUse
- **PR 创建**：记录 PR URL 和 GitHub Actions 状态
- **Prettier**：编辑后自动格式化 JS/TS 文件
- **TypeScript 检查**：编辑 .ts/.tsx 文件后执行 tsc
- **console.log 警告**：警告编辑文件中的 console.log

### Stop
- **console.log 审计**：会话结束前检查所有修改文件中的 console.log

## 自动接受权限

谨慎使用：
- 对可信、定义明确的计划启用
- 对探索性工作禁用
- 绝不使用 dangerously-skip-permissions 参数
- 改为在 `~/.claude.json` 中配置 `allowedTools`

## TodoWrite 最佳实践

使用 TodoWrite 工具来：
- 追踪多步骤任务的进度
- 验证对指令的理解
- 支持即时调整
- 显示细粒度实现步骤

待办清单可揭示：
- 顺序错误的步骤
- 遗漏的项目
- 多余的不必要项目
- 错误的粒度
- 误解的需求
