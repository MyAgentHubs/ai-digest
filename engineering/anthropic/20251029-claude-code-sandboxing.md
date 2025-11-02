# 超越权限提示：让 Claude Code 更安全、更自主

**原文链接**: https://www.anthropic.com/engineering/claude-code-sandboxing
**发布日期**: 2025 年 10 月 20 日

当 AI Agent 可以执行代码和修改文件时，安全性至关重要。但传统的权限提示方法——每次操作都要求用户批准——会导致令人沮丧的体验，并减缓 Agent 的效率。

本文介绍 Claude Code 如何通过沙箱（sandboxing）超越简单的权限提示，创建一个既安全又能让 Agent 自主工作的环境。

## 权限提示的问题

在早期版本的 Claude Code 中，我们使用传统的权限提示方法：

```
Claude 想要：
- 读取 /home/user/project/config.json
- 写入 /home/user/project/output.txt
- 执行 npm install

[允许] [拒绝]
```

虽然这提供了安全性，但也带来了几个问题：

### 1. 提示疲劳

用户很快对不断的提示感到疲倦，开始不假思索地点击"允许"，这削弱了安全性。

### 2. 中断工作流程

每次提示都会打断 Agent 的工作，将本该是流畅的任务变成了断断续续的体验。

### 3. 缺乏上下文

用户通常不确定特定权限请求是否合理，因为他们缺少 Agent 决策的完整上下文。

### 4. 效率低下

对于需要数十或数百次文件操作的复杂任务，权限提示变得不可行。

## 沙箱：更安全、更自主的方法

沙箱采用不同的方法：与其每次都询问权限，不如提前定义 Agent 可以访问的边界。

### 核心概念

沙箱创建了一个受限环境，其中：

1. **预定义访问**：你提前指定 Agent 可以访问哪些目录和资源
2. **隔离执行**：Agent 在受控环境中运行，无法访问边界之外的内容
3. **自主操作**：在沙箱内，Agent 可以自由工作，无需不断的提示

### 工作原理

```typescript
// 配置沙箱
const sandbox = {
  filesystem: {
    // 只读访问
    readOnly: [
      '/home/user/project/src',
      '/home/user/project/tests'
    ],
    // 读写访问
    readWrite: [
      '/home/user/project/output',
      '/home/user/project/.cache'
    ]
  },
  network: {
    // 允许的域
    allowedDomains: [
      'api.github.com',
      'registry.npmjs.org'
    ]
  },
  processes: {
    // 允许的命令
    allowedCommands: [
      'npm',
      'git',
      'node'
    ]
  }
}
```

### 文件系统沙箱

Claude Code 的文件系统沙箱提供细粒度控制：

#### 只读访问

```json
{
  "readOnly": [
    "/home/user/project/src",
    "/home/user/project/config"
  ]
}
```

Agent 可以：
- ✅ 读取这些目录中的文件
- ✅ 列出目录内容
- ✅ 检查文件元数据

Agent 不能：
- ❌ 修改现有文件
- ❌ 创建新文件
- ❌ 删除文件

#### 读写访问

```json
{
  "readWrite": [
    "/home/user/project/output",
    "/home/user/project/tmp"
  ]
}
```

Agent 拥有完全访问权限：
- ✅ 读取文件
- ✅ 创建文件
- ✅ 修改文件
- ✅ 删除文件

#### 路径解析

沙箱智能处理路径：

```typescript
// 绝对路径
readOnly: ['/home/user/project/src']

// 相对路径（相对于项目根目录）
readOnly: ['./src', './tests']

// 通配符模式
readOnly: ['./src/**/*.ts']

// 排除模式
readOnly: ['./src'],
exclude: ['./src/generated']
```

### 网络沙箱

控制 Agent 可以访问的网络资源：

```json
{
  "network": {
    "allowedDomains": [
      "api.github.com",
      "registry.npmjs.org"
    ],
    "blockedDomains": [
      "*.amazonaws.com"
    ],
    "allowedPorts": [80, 443]
  }
}
```

这可以防止：
- 未经授权的数据泄露
- 访问内部网络资源
- 意外的云服务成本

### 进程沙箱

限制 Agent 可以执行的命令：

```json
{
  "processes": {
    "allowedCommands": ["npm", "git", "node"],
    "blockedCommands": ["rm", "dd", "curl"],
    "environment": {
      "NODE_ENV": "development",
      "MAX_MEMORY": "512M"
    }
  }
}
```

## 实际使用案例

### 案例 1：代码审查 Agent

```json
{
  "name": "code-reviewer",
  "sandbox": {
    "filesystem": {
      "readOnly": [
        "./src",
        "./tests",
        "./.git"
      ],
      "readWrite": [
        "./review-comments.md"
      ]
    },
    "network": {
      "allowedDomains": ["api.github.com"]
    },
    "processes": {
      "allowedCommands": ["git"]
    }
  }
}
```

这个沙箱允许 Agent：
- ✅ 读取源代码和测试
- ✅ 检查 git 历史
- ✅ 编写审查评论
- ✅ 访问 GitHub API

但防止它：
- ❌ 修改源代码
- ❌ 访问其他 API
- ❌ 执行危险命令

### 案例 2：文档生成器

```json
{
  "name": "doc-generator",
  "sandbox": {
    "filesystem": {
      "readOnly": ["./src", "./examples"],
      "readWrite": ["./docs"]
    },
    "network": {
      "allowedDomains": ["cdn.jsdelivr.net"]
    },
    "processes": {
      "allowedCommands": ["node"]
    }
  }
}
```

### 案例 3：测试运行器

```json
{
  "name": "test-runner",
  "sandbox": {
    "filesystem": {
      "readOnly": ["./src", "./tests"],
      "readWrite": ["./coverage", "./.test-cache"]
    },
    "network": {
      "allowedDomains": []  // 无网络访问
    },
    "processes": {
      "allowedCommands": ["npm", "node"],
      "environment": {
        "NODE_ENV": "test"
      }
    }
  }
}
```

## 性能影响

我们在内部测试中衡量了沙箱的影响：

### 权限提示减少

**之前（权限提示）**：
- 平均每个任务 47 次提示
- 用户需要手动批准每个操作
- 任务被不断中断

**之后（沙箱）**：
- 每个任务 0 次提示（减少 100%）
- Agent 在定义的边界内自主工作
- 流畅、不间断的执行

### 任务完成时间

| 任务类型 | 使用权限提示 | 使用沙箱 | 改进 |
|---------|-------------|---------|------|
| 代码重构 | 12.3 分钟 | 3.8 分钟 | **69% 更快** |
| 测试生成 | 8.7 分钟 | 2.1 分钟 | **76% 更快** |
| 文档更新 | 15.2 分钟 | 4.5 分钟 | **70% 更快** |

### 用户满意度

- **权限提示方式**：3.2/5 星
- **沙箱方式**：4.6/5 星
- **改进**：+44%

## 安全保证

沙箱提供了强大的安全保证：

### 1. 路径遍历保护

```python
# Agent 尝试访问：
path = "../../etc/passwd"

# 沙箱将其解析为：
resolved = "/home/user/project/../../etc/passwd"
          = "/etc/passwd"

# 检查边界：
if not is_within_sandbox(resolved):
    raise PermissionError("Access denied: outside sandbox")
```

### 2. 符号链接保护

```bash
# 恶意尝试：
ln -s /etc/passwd ./src/config.txt

# 沙箱检测符号链接目标：
target = os.readlink('./src/config.txt')  # /etc/passwd
if not is_within_sandbox(target):
    raise PermissionError("Symlink target outside sandbox")
```

### 3. 进程隔离

```python
# Agent 尝试：
subprocess.run(['rm', '-rf', '/'])

# 沙箱拦截：
if command not in allowed_commands:
    raise PermissionError(f"Command not allowed: {command}")
```

### 4. 网络隔离

```python
# Agent 尝试：
requests.get('https://internal.company.com/api')

# 沙箱检查：
domain = extract_domain(url)
if domain not in allowed_domains:
    raise PermissionError(f"Domain not allowed: {domain}")
```

## 最佳实践

### 1. 最小权限原则

只授予 Agent 完成任务所需的最小访问权限：

```json
// ❌ 过于宽松
{
  "readWrite": ["/"]  // 整个文件系统！
}

// ✅ 适当限制
{
  "readOnly": ["./src"],
  "readWrite": ["./output"]
}
```

### 2. 使用特定沙箱配置

为不同任务创建专门的沙箱：

```json
{
  "sandboxes": {
    "code-review": {
      "filesystem": {
        "readOnly": ["./src", "./tests"],
        "readWrite": ["./review.md"]
      }
    },
    "refactoring": {
      "filesystem": {
        "readOnly": ["./tests"],
        "readWrite": ["./src"]
      }
    }
  }
}
```

### 3. 审计和监控

即使使用沙箱，也要记录 Agent 的操作：

```json
{
  "audit": {
    "logLevel": "detailed",
    "logPath": "./logs/agent-activity.log",
    "alertOnPatterns": [
      "delete",
      "rm",
      "DROP TABLE"
    ]
  }
}
```

### 4. 测试沙箱配置

在生产中使用之前验证沙箱：

```bash
# 测试模式
claude-code --sandbox-test ./sandbox-config.json

# 输出：
✅ Filesystem boundaries validated
✅ Network restrictions tested
✅ Process controls verified
⚠️  Warning: readWrite includes sensitive directory ./config
```

### 5. 文档化沙箱策略

清楚地记录为什么做出特定选择：

```json
{
  "filesystem": {
    "readOnly": ["./src"],
    // 原因：防止意外修改生产代码
    "readWrite": ["./test-output"]
    // 原因：允许创建测试报告
  }
}
```

## 与容器化的比较

你可能想知道沙箱与 Docker 容器有何不同：

| 特性 | Claude Code 沙箱 | Docker 容器 |
|-----|------------------|------------|
| **设置复杂度** | 简单配置文件 | 需要 Dockerfile |
| **启动时间** | <100ms | 1-5 秒 |
| **资源开销** | 最小 | 中等 |
| **细粒度控制** | 文件/网络/进程级别 | 容器级别 |
| **与主机集成** | 无缝 | 需要卷映射 |
| **学习曲线** | 低 | 中等 |

沙箱是轻量级的，专门为 AI Agent 使用场景设计。

## 开始使用

### 基本沙箱配置

创建 `.claude-code/sandbox.json`：

```json
{
  "filesystem": {
    "readOnly": ["./src", "./tests"],
    "readWrite": ["./output"]
  },
  "network": {
    "allowedDomains": ["api.github.com"]
  },
  "processes": {
    "allowedCommands": ["npm", "git", "node"]
  }
}
```

### 启用沙箱

```bash
# 使用默认沙箱
claude-code --sandbox

# 使用自定义配置
claude-code --sandbox-config ./my-sandbox.json

# 在交互模式中启用
claude-code
> enable sandbox ./sandbox.json
✅ Sandbox enabled
```

### 验证沙箱

```bash
# 检查沙箱状态
claude-code --sandbox-status

# 输出：
Sandbox: ACTIVE
Config: ./.claude-code/sandbox.json
Boundaries:
  Read-only: ./src, ./tests
  Read-write: ./output
  Network: api.github.com
  Commands: npm, git, node
```

### 调试沙箱问题

```bash
# 详细模式查看所有检查
claude-code --sandbox --verbose

# 输出：
[Sandbox] Checking file access: ./src/main.ts
[Sandbox] ✅ Within read-only boundary
[Sandbox] Checking network access: api.github.com
[Sandbox] ✅ Domain allowed
[Sandbox] Checking command: npm install
[Sandbox] ✅ Command allowed
```

## 高级功能

### 动态沙箱

根据任务类型调整沙箱：

```json
{
  "dynamicSandbox": {
    "rules": [
      {
        "if": "task.type == 'review'",
        "then": {
          "filesystem": {
            "readOnly": ["./src"],
            "readWrite": []
          }
        }
      },
      {
        "if": "task.type == 'refactor'",
        "then": {
          "filesystem": {
            "readWrite": ["./src"]
          }
        }
      }
    ]
  }
}
```

### 临时权限提升

对于某些操作，允许临时超出沙箱：

```json
{
  "temporaryEscalation": {
    "enabled": true,
    "requiresConfirmation": true,
    "maxDuration": "5m",
    "allowedFor": ["npm install"]
  }
}
```

### 沙箱继承

创建可重用的基础配置：

```json
// base-sandbox.json
{
  "filesystem": {
    "readOnly": ["./src", "./tests"]
  }
}

// specific-sandbox.json
{
  "extends": "./base-sandbox.json",
  "filesystem": {
    "readWrite": ["./output"]
  }
}
```

## 未来方向

我们正在探索的沙箱增强功能：

### 1. 自适应沙箱

根据 Agent 行为自动调整边界：

```python
# 如果 Agent 从不访问某些目录
# 自动收紧边界
if not accessed_in_last_10_tasks('./docs'):
    remove_from_sandbox('./docs')
```

### 2. 安全策略模板

为常见场景预配置的沙箱：

```bash
# 使用预定义模板
claude-code --sandbox-template web-development
claude-code --sandbox-template data-analysis
claude-code --sandbox-template testing
```

### 3. 团队级沙箱策略

组织范围的沙箱配置：

```json
// .claude-code/org-policy.json
{
  "organization": "acme-corp",
  "defaultSandbox": {
    "filesystem": {
      "neverAllow": ["/etc", "/var"],
      "alwaysAllow": ["./workspace"]
    }
  }
}
```

### 4. 沙箱分析

了解 Agent 如何使用权限：

```bash
claude-code --sandbox-report

# 输出：
Sandbox Usage Report (Last 30 days)
====================================
Most accessed: ./src (1,234 operations)
Never accessed: ./legacy (0 operations)
Recommendations:
  - Consider removing ./legacy from sandbox
  - Add ./api to read-only (accessed 89 times)
```

## 关键要点

1. ✅ **沙箱 > 权限提示**：提前定义边界而不是不断询问
2. ✅ **细粒度控制**：文件系统、网络和进程级别的限制
3. ✅ **显著的性能提升**：任务完成速度提高 70%，权限提示减少 100%
4. ✅ **强大的安全性**：路径遍历、符号链接和进程注入保护
5. ✅ **易于设置**：简单的 JSON 配置文件
6. ✅ **灵活**：针对不同任务的特定沙箱
7. ✅ **可审计**：记录所有 Agent 操作

## 致谢

Claude Code 沙箱系统的开发是跨职能协作的结果。特别感谢：

- 安全团队设计沙箱架构
- 工程团队实现和优化
- 产品团队定义用户体验
- Beta 测试人员提供宝贵反馈

我们还从以下开源项目中获得了灵感：
- [Firejail](https://firejail.wordpress.com/) - Linux 沙箱
- [Bubblewrap](https://github.com/containers/bubblewrap) - 容器沙箱
- [Sandboxie](https://www.sandboxie.com/) - Windows 沙箱

## 资源

- [Claude Code 文档](https://docs.anthropic.com/claude-code)
- [沙箱配置参考](https://docs.anthropic.com/claude-code/sandbox)
- [安全最佳实践](https://docs.anthropic.com/claude-code/security)
- [示例沙箱配置](https://github.com/anthropics/claude-code-examples/tree/main/sandboxes)

---

**整理者**: Panda
**整理日期**: 2025 年 10 月 29 日
