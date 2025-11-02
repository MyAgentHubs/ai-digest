# Desktop Extensions：Claude Desktop 一键安装 MCP 服务器

**发布日期：2025年6月26日**

Desktop Extensions 让安装 MCP（模型上下文协议）服务器就像点击按钮一样简单。我们分享了技术架构和创建优质扩展的技巧。

## 最新更新

**文件扩展名更新（2025年9月11日）**

Claude Desktop Extensions 现使用 `.mcpb`（MCP Bundle）文件扩展名，取代 `.dxt`。现有 `.dxt` 扩展仍可继续使用，但建议开发者在新扩展中采用 `.mcpb`。所有功能保持不变，仅为命名规范更新。

---

## 解决 MCP 安装问题

去年发布模型上下文协议后，我们看到开发者构建了令人惊艳的本地服务器。然而用户反馈一致：安装过程过于复杂。用户需要开发者工具、手动编辑配置文件、经常陷入依赖问题。

本地 MCP 服务器为 Claude Desktop 用户解锁强大功能。它们可与本地应用交互、访问私密数据、集成开发工具，同时保持数据在用户机器上。但当前安装流程存在显著障碍：

- **需要开发者工具**：用户需安装 Node.js、Python 等运行时
- **手动配置**：每个服务器需编辑 JSON 配置文件
- **依赖管理**：用户需解决包冲突和版本不匹配
- **无发现机制**：找到有用的 MCP 服务器需在 GitHub 搜索
- **更新复杂**：保持服务器最新需手动重装

这些摩擦点导致 MCP 服务器对非技术用户基本无法使用。

## 引入 Desktop Extensions

Desktop Extensions（`.mcpb` 文件）通过将整个 MCP 服务器及所有依赖打包成单个可安装包来解决这些问题。

**之前的流程：**

```bash
# 先安装 Node.js
npm install -g @example/mcp-server
# 手动编辑 ~/.claude/claude_desktop_config.json
# 重启 Claude Desktop
# 祈祷它能工作
```

**现在的流程：**

1. 下载 `.mcpb` 文件
2. 双击用 Claude Desktop 打开
3. 点击"安装"

就这么简单。无需终端、无需配置文件、无需依赖冲突。

## 架构概览

Desktop Extension 是包含本地 MCP 服务器的 zip 存档，以及描述 Claude Desktop 和其他支持桌面扩展的应用所需所有信息的 `manifest.json`。

```
extension.mcpb (ZIP 存档)
├── manifest.json         # 扩展元数据和配置
├── server/               # MCP 服务器实现
│   └── [server files]    # 服务器文件
├── dependencies/         # 所有需要的包/库
└── icon.png             # 可选：扩展图标

# Node.js 扩展示例
extension.mcpb
├── manifest.json         # 必需：扩展元数据和配置
├── server/               # 服务器文件
│   └── index.js          # 主入口点
├── node_modules/         # 打包的依赖
├── package.json          # 可选：NPM 包定义
└── icon.png              # 可选：扩展图标

# Python 扩展示例
extension.mcpb (ZIP 文件)
├── manifest.json         # 必需：扩展元数据和配置
├── server/               # 服务器文件
│   ├── main.py           # 主入口点
│   └── utils.py          # 其他模块
├── lib/                  # 打包的 Python 包
├── requirements.txt      # 可选：Python 依赖列表
└── icon.png              # 可选：扩展图标
```

Desktop Extension 中仅有 `manifest.json` 是必需的。Claude Desktop 处理所有复杂性：

- **内置运行时**：Claude Desktop 附带 Node.js，消除外部依赖
- **自动更新**：新版本可用时扩展自动更新
- **安全密钥**：敏感配置（如 API 密钥）存储在操作系统密钥环中

清单包含人类可读的信息（名称、描述、作者）、功能声明（工具、提示）、用户配置和运行时要求。大多数字段可选，最小版本相当简洁。实际上，我们期望所有三种支持的扩展类型（Node.js、Python 和经典二进制/可执行文件）都包含文件：

```json
{
  "mcpb_version": "0.1",              // 此清单符合的 MCPB 规范版本
  "name": "my-extension",             // 机器可读名称（用于 CLI、API）
  "version": "1.0.0",                 // 扩展的语义版本
  "description": "A simple MCP extension", // 扩展功能简述
  "author": {                         // 作者信息（必需）
    "name": "Extension Author"        // 作者名称（必需字段）
  },
  "server": {                         // 服务器配置（必需）
    "type": "node",                   // 服务器类型："node"、"python" 或 "binary"
    "entry_point": "server/index.js", // 主服务器文件路径
    "mcp_config": {                   // MCP 服务器配置
      "command": "node",              // 运行服务器的命令
      "args": [                       // 传递给命令的参数
        "${__dirname}/server/index.js" // ${__dirname} 替换为扩展目录
      ]
    }
  }
}
```

[清单规范](https://github.com/anthropics/dxt/blob/main/MANIFEST.md)中提供了多项便利选项，旨在简化本地 MCP 服务器的安装和配置。服务器配置对象可定义为支持模板字面量形式的用户定义配置和平台特定覆盖。扩展开发者可详细定义想要收集的配置类型。

以下清单示例展示清单如何协助配置。开发者声明用户需提供 `api_key`。Claude 在用户提供该值前不会启用扩展，自动将其保存在操作系统密钥库中，并在启动服务器时透明替换 `${user_config.api_key}`。类似地，`${__dirname}` 会替换为扩展的完整解包目录路径。

```json
{
  "mcpb_version": "0.1",
  "name": "my-extension",
  "version": "1.0.0",
  "description": "A simple MCP extension",
  "author": {
    "name": "Extension Author"
  },
  "server": {
    "type": "node",
    "entry_point": "server/index.js",
    "mcp_config": {
      "command": "node",
      "args": ["${__dirname}/server/index.js"],
      "env": {
        "API_KEY": "${user_config.api_key}"
      }
    }
  },
  "user_config": {
    "api_key": {
      "type": "string",
      "title": "API Key",
      "description": "Your API key for authentication",
      "sensitive": true,
      "required": true
    }
  }
}
```

包含大多数可选字段的完整 `manifest.json` 可能如下所示：

```json
{
  "mcpb_version": "0.1",
  "name": "My MCP Extension",
  "display_name": "My Awesome MCP Extension",
  "version": "1.0.0",
  "description": "A brief description of what this extension does",
  "long_description": "A detailed description that can include multiple paragraphs explaining the extension's functionality, use cases, and features. It supports basic markdown.",
  "author": {
    "name": "Your Name",
    "email": "yourname@example.com",
    "url": "https://your-website.com"
  },
  "repository": {
    "type": "git",
    "url": "https://github.com/your-username/my-mcp-extension"
  },
  "homepage": "https://example.com/my-extension",
  "documentation": "https://docs.example.com/my-extension",
  "support": "https://github.com/your-username/my-extension/issues",
  "icon": "icon.png",
  "screenshots": [
    "assets/screenshots/screenshot1.png",
    "assets/screenshots/screenshot2.png"
  ],
  "server": {
    "type": "node",
    "entry_point": "server/index.js",
    "mcp_config": {
      "command": "node",
      "args": ["${__dirname}/server/index.js"],
      "env": {
        "ALLOWED_DIRECTORIES": "${user_config.allowed_directories}"
      }
    }
  },
  "tools": [
    {
      "name": "search_files",
      "description": "Search for files in a directory"
    }
  ],
  "prompts": [
    {
      "name": "poetry",
      "description": "Have the LLM write poetry",
      "arguments": ["topic"],
      "text": "Write a creative poem about the following topic: ${arguments.topic}"
    }
  ],
  "tools_generated": true,
  "keywords": ["api", "automation", "productivity"],
  "license": "MIT",
  "compatibility": {
    "claude_desktop": ">=1.0.0",
    "platforms": ["darwin", "win32", "linux"],
    "runtimes": {
      "node": ">=16.0.0"
    }
  },
  "user_config": {
    "allowed_directories": {
      "type": "directory",
      "title": "Allowed Directories",
      "description": "Directories the server can access",
      "multiple": true,
      "required": true,
      "default": ["${HOME}/Desktop"]
    },
    "api_key": {
      "type": "string",
      "title": "API Key",
      "description": "Your API key for authentication",
      "sensitive": true,
      "required": false
    },
    "max_file_size": {
      "type": "number",
      "title": "Maximum File Size (MB)",
      "description": "Maximum file size to process",
      "default": 10,
      "min": 1,
      "max": 100
    }
  }
}
```

要查看扩展和清单，请参考 [MCPB 仓库中的示例](https://github.com/anthropics/dxt/tree/main/examples)。

`manifest.json` 中所有必需和可选字段的完整规范可在我们的[开源工具链](https://github.com/anthropics/dxt/blob/main/MANIFEST.md)中找到。

## 构建第一个扩展

让我们逐步讲解如何将现有 MCP 服务器打包为 Desktop Extension。我们将以简单的文件系统服务器为例。

### 步骤 1：创建清单

首先为服务器初始化清单：

```bash
npx @anthropic-ai/mcpb init
```

此交互式工具会询问关于服务器的问题并生成完整的 `manifest.json`。如果想快速生成最基础的 `manifest.json`，可使用 `--yes` 参数运行命令。

### 步骤 2：处理用户配置

如果服务器需要用户输入（如 API 密钥或允许的目录），在清单中声明：

```json
"user_config": {
  "allowed_directories": {
    "type": "directory",
    "title": "Allowed Directories",
    "description": "Directories the server can access",
    "multiple": true,
    "required": true,
    "default": ["${HOME}/Documents"]
  }
}
```

Claude Desktop 会：

- 显示用户友好的配置界面
- 启用扩展前验证输入
- 安全地存储敏感值
- 根据开发者配置将配置传递给服务器，作为参数或环境变量

下例中，我们将用户配置作为环境变量传递，但也可作为参数：

```json
"server": {
  "type": "node",
  "entry_point": "server/index.js",
  "mcp_config": {
    "command": "node",
    "args": ["${__dirname}/server/index.js"],
    "env": {
      "ALLOWED_DIRECTORIES": "${user_config.allowed_directories}"
    }
  }
}
```

### 步骤 3：打包扩展

将所有内容打包为 `.mcpb` 文件：

```bash
npx @anthropic-ai/mcpb pack
```

此命令会：

1. 验证清单
2. 生成 `.mcpb` 存档

### 步骤 4：本地测试

将 `.mcpb` 文件拖入 Claude Desktop 的"设置"窗口。你将看到：

- 关于扩展的人类可读信息
- 必需权限和配置
- 简单的"安装"按钮

## 高级功能

### 跨平台支持

扩展可适应不同操作系统：

```json
"server": {
  "type": "node",
  "entry_point": "server/index.js",
  "mcp_config": {
    "command": "node",
    "args": ["${__dirname}/server/index.js"],
    "platforms": {
      "win32": {
        "command": "node.exe",
        "env": {
          "TEMP_DIR": "${TEMP}"
        }
      },
      "darwin": {
        "env": {
          "TEMP_DIR": "${TMPDIR}"
        }
      }
    }
  }
}
```

### 动态配置

使用模板字面量获取运行时值：

- `${__dirname}`：扩展安装目录
- `${user_config.key}`：用户提供的配置
- `${HOME}, ${TEMP}`：系统环境变量

### 功能声明

提前帮助用户了解功能：

```json
"tools": [
  {
    "name": "read_file",
    "description": "Read contents of a file"
  }
],
"prompts": [
  {
    "name": "code_review",
    "description": "Review code for best practices",
    "arguments": ["file_path"]
  }
]
```

## 扩展目录

我们推出了 Claude Desktop 内置的精选扩展目录。用户可浏览、搜索和一键安装——无需搜索 GitHub 或审查代码。

虽然我们期望 Desktop Extension 规范和 Claude macOS/Windows 实现会随时间演进，但我们期待看到扩展以创意方式拓展 Claude 功能的众多方式。

提交扩展：

1. 确保遵循提交表单中的指南
2. 在 Windows 和 macOS 上测试
3. [提交扩展](https://docs.google.com/forms/d/14_Dmcig4z8NeRMB_e7TOyrKzuZ88-BLYdLvS6LPhiZU/edit)
4. 我们的团队审查质量和安全性

## 构建开放生态系统

我们致力于围绕 MCP 服务器的开放生态系统，相信其被多个应用和服务通用采用的能力已惠及社区。按照这一承诺，我们开源了 Desktop Extension 规范、工具链，以及 Claude macOS 和 Windows 用于实现自身 Desktop Extensions 支持的模式和关键函数。我们希望 MCPB 格式不仅使 Claude 的本地 MCP 服务器更可移植，也使其他 AI 桌面应用更便携。

我们开源的内容：

- 完整 MCPB 规范
- 打包和验证工具
- 参考实现代码
- TypeScript 类型和模式

这意味着：

- **对于 MCP 服务器开发者**：打包一次，在任何支持 MCPB 的地方运行
- **对于应用开发者**：无需从头构建即可添加扩展支持
- **对于用户**：在所有启用 MCP 的应用中获得一致体验

规范和工具链故意版本化为 0.1，因为我们期待与更大社区合作演进和改变格式。我们期待听到你的想法。

## 安全与企业考量

我们理解扩展带来新的安全考量，特别是对企业而言。我们在 Desktop Extensions 预览版中内置了多项保障：

### 对用户

- 敏感数据保留在操作系统密钥环中
- 自动更新
- 能够审计已安装扩展

### 对企业

- 组策略（Windows）和移动设备管理（macOS）支持
- 能够预安装已批准扩展
- 将特定扩展或发布者加入黑名单
- 完全禁用扩展目录
- 部署私有扩展目录

有关如何在组织内管理扩展的更多信息，请参阅我们的[文档](https://support.anthropic.com/en/articles/10949351-getting-started-with-model-context-protocol-mcp-on-claude-for-desktop)。

## 入门

准备好构建自己的扩展了吗？以下是开始的方法：

**对于 MCP 服务器开发者**：查看我们的[开发者文档](https://github.com/anthropics/dxt)——或在本地 MCP 服务器目录中运行以下命令直接上手：

```bash
npm install -g @anthropic-ai/mcpb
mcpb init
mcpb pack
```

**对于 Claude Desktop 用户**：更新到最新版本，在设置中查找扩展部分

**对于企业**：查看我们的企业文档了解部署选项

## 使用 Claude Code 构建

在 Anthropic 内部，我们发现 Claude 可以以最少的干预构建扩展。如果你也想使用 Claude Code，我们建议你简要说明想要扩展做什么，然后将以下上下文添加到提示：

```
我想将其构建为 Desktop Extension（简称"MCPB"）。请按照以下步骤：

1. **彻底阅读规范：**
   - https://github.com/anthropics/mcpb/blob/main/README.md - MCPB 架构概览、功能和集成模式
   - https://github.com/anthropics/mcpb/blob/main/MANIFEST.md - 完整扩展清单结构和字段定义
   - https://github.com/anthropics/mcpb/tree/main/examples - 参考实现，包括"Hello World"示例

2. **创建适当的扩展结构：**
   - 按照 MANIFEST.md 规范生成有效的 manifest.json
   - 使用 @modelcontextprotocol/sdk 实现 MCP 服务器，包含适当的工具定义
   - 包含适当的错误处理和超时管理

3. **遵循最佳开发实践：**
   - 通过 stdio 传输实现适当的 MCP 协议通信
   - 使用清晰的模式、验证和一致的 JSON 响应构造工具
   - 利用此扩展将在本地运行的事实
   - 添加适当的日志和调试功能
   - 包含适当的文档和设置说明

4. **测试考量：**
   - 验证所有工具调用都返回正确结构的响应
   - 验证清单加载正确，主机集成有效

生成可立即测试的完整、生产就绪代码。专注于防御性编程、清晰的错误消息和遵循确切的 MCPB 规范以确保生态系统兼容性。
```

## 结论

Desktop Extensions 代表用户与本地 AI 工具交互方式的根本转变。通过消除安装摩擦，我们使强大的 MCP 服务器对每个人都可访问——而不仅仅是开发者。

在 Anthropic 内部，我们使用桌面扩展来共享高度实验性的 MCP 服务器——有些有趣，有些实用。一个团队进行了实验，看看我们的模型在直接连接到 GameBoy 时能走多远，类似于我们的["Claude 玩精灵宝可梦"研究](https://www.anthropic.com/news/visible-extended-thinking)。我们使用 Desktop Extensions 打包了单个扩展，打开了流行的 [PyBoy](https://github.com/Baekalfen/PyBoy) GameBoy 模拟器并让 Claude 控制。我们相信存在无数机会将模型的功能连接到用户本地机器上已有的工具、数据和应用。

![显示 PyBoy MCP 和超级马里奥乐园开始画面的桌面](https://cdn.sanity.io/images/4zrzovbb/website/d48f3ea1218a4b90450b9ab8134fa0e24db5a167-720x542.png)

我们迫不及待想看你构建什么。将我们带给数千个 MCP 服务器的创意现在可以通过一次点击触及数百万用户。准备好分享你的 MCP 服务器了吗？[提交扩展以供审查](https://forms.gle/tyiAZvch1kDADKoP9)。

## 想了解更多？

通过 Anthropic Academy 的课程掌握 API 开发、模型上下文协议和 Claude Code。完成后获得证书。

[探索课程](https://anthropic.skilljar.com/)
