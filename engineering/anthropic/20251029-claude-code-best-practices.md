# Claude Code：Agent 编码的最佳实践

**原文链接**: https://www.anthropic.com/engineering/claude-code-best-practices
**发布日期**: 2025 年 4 月 18 日

我们最近[发布了 Claude Code](https://www.anthropic.com/news/claude-3-7-sonnet)，这是一个用于 Agent 编码的命令行工具。作为一个研究项目开发，Claude Code 为 Anthropic 的工程师和研究人员提供了一种更原生的方式，将 Claude 集成到他们的编码工作流程中。

Claude Code 设计理念是低层次且不带偏见，提供接近原始模型的访问，而不强制特定的工作流程。这种设计哲学创造了一个灵活、可定制、可脚本化且安全的强大工具。虽然强大，但这种灵活性为不熟悉 Agent 编码工具的工程师带来了学习曲线——至少在他们开发出自己的最佳实践之前。

本文概述了已被证明有效的通用模式，无论是 Anthropic 的内部团队还是在各种代码库、语言和环境中使用 Claude Code 的外部工程师。这份清单中的内容都不是一成不变的，也不是普遍适用的；将这些建议视为起点。我们鼓励你进行实验，找到最适合你的方法！

*想了解更多详细信息？我们在 [claude.ai/code](https://claude.ai/code) 的综合文档涵盖了本文中提到的所有功能，并提供了额外的示例、实现细节和高级技术。*

## 1. 自定义你的设置

Claude Code 是一个自动将上下文拉入提示的 Agent 编码助手。这种上下文收集会消耗时间和 token，但你可以通过环境调优来优化它。

### a. 创建 `CLAUDE.md` 文件

`CLAUDE.md` 是一个特殊文件，Claude 在开始对话时会自动将其拉入上下文。这使它成为记录以下内容的理想位置：

- 常见的 bash 命令
- 核心文件和实用函数
- 代码风格指南
- 测试说明
- 仓库礼仪（例如，分支命名、merge vs rebase 等）
- 开发环境设置（例如，pyenv use，哪些编译器有效）
- 项目特有的任何意外行为或警告
- 你希望 Claude 记住的其他信息

`CLAUDE.md` 文件没有固定格式。我们建议保持简洁且易读。例如：

```markdown
# Bash 命令

- 运行测试：`npm test`
- 构建项目：`npm run build`
- 启动开发服务器：`npm run dev`

# 核心文件

- `src/utils/api.ts` - API 辅助函数
- `src/hooks/useAuth.ts` - 认证 hook

# 代码风格

- 使用 TypeScript strict 模式
- Prefer functional components
- 使用 ESLint 和 Prettier
```

### b. 配置 `.gitignore`

确保将 Claude Code 的工作文件添加到 `.gitignore`：

```
# Claude Code
.claude/
.claude-code/
```

### c. 使用项目特定的配置

Claude Code 支持项目级配置文件（`.claude-code/config.json`），可以自定义：

- 使用的模型
- 上下文窗口大小
- 自定义提示前缀
- 工具可用性

## 2. 给 Claude 更多工具

Claude Code 提供了一组核心工具（如 Bash、Read、Edit、Write），但你可以扩展这些功能。

### a. 创建自定义工具

通过 MCP (Model Context Protocol) 添加自定义工具：

```typescript
// 示例：自定义数据库查询工具
{
  "name": "query_db",
  "description": "查询项目数据库",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "SQL 查询"
      }
    },
    "required": ["query"]
  }
}
```

### b. 使用 Skills

Skills 是可重用的工作流程，可以在多个项目中共享。创建 skills 来自动化常见任务：

- 代码审查
- 测试生成
- 文档更新
- 部署流程

## 3. 尝试常见工作流程

以下是一些经过验证的工作流程：

### a. 迭代开发

```
1. 描述你想要构建的功能
2. 让 Claude 创建初始实现
3. 运行测试并分享结果
4. 迭代改进
```

### b. 调试

```
1. 分享错误消息或堆栈跟踪
2. 让 Claude 分析问题
3. 应用建议的修复
4. 验证修复有效
```

### c. 代码审查

```
1. 让 Claude 审查你的更改
2. 讨论建议
3. 应用改进
4. 再次审查
```

## 4. 优化你的工作流程

### a. 使用批处理命令

通过批处理相关命令减少往返：

```bash
# 而不是多个独立命令
claude "read file1" && claude "read file2" && claude "compare them"

# 使用单个复合请求
claude "read file1 and file2, then compare them"
```

### b. 利用上下文记忆

Claude Code 在会话中保持上下文。利用这一点：

```
# 第一个命令
claude "analyze the auth system"

# 后续命令（Claude 记得之前的分析）
claude "now refactor it for better security"
```

### c. 使用 Plan 模式

对于复杂任务，使用 plan 模式获取执行前的计划：

```bash
claude --plan "refactor the entire API layer"
```

## 5. 使用 Headless 模式自动化基础设施

Headless 模式允许你在 CI/CD 管道中运行 Claude Code：

```bash
# 在 CI 中自动生成文档
claude --headless "update all API documentation"

# 自动代码审查
claude --headless "review PR #123"
```

## 6. 安全最佳实践

### a. 审查所有更改

始终审查 Claude 建议的更改，特别是：

- 安全相关代码
- 数据库迁移
- 配置文件
- 生产部署

### b. 使用沙箱

在沙箱环境中测试更改：

```bash
claude --sandbox "test this risky change"
```

### c. 限制权限

使用配置限制 Claude 可以访问的内容：

```json
{
  "permissions": {
    "write": ["src/**", "tests/**"],
    "execute": ["npm", "git"]
  }
}
```

## 7. 协作技巧

### a. 共享会话

使用 `claude export` 共享会话：

```bash
claude export session.json
# 队友可以导入并继续
claude import session.json
```

### b. 记录决策

在 `CLAUDE.md` 中记录重要决策：

```markdown
# 架构决策

## 2025-04-18：选择 PostgreSQL 而不是 MongoDB

理由：需要复杂查询和事务支持
```

## 结论

Claude Code 是一个强大而灵活的工具。这些最佳实践应该帮助你更有效地使用它，但不要害怕实验并找到最适合你团队的工作流程。

关键要点：

1. ✅ 使用 `CLAUDE.md` 提供项目上下文
2. ✅ 扩展工具集以满足你的需求
3. ✅ 开发迭代工作流程
4. ✅ 优化性能和成本
5. ✅ 在 CI/CD 中使用 headless 模式
6. ✅ 始终审查和测试更改
7. ✅ 与团队分享知识

想了解更多？访问 [claude.ai/code](https://claude.ai/code) 获取完整文档、教程和社区资源。

---

**整理者**: Panda
**整理日期**: 2025 年 10 月 29 日
