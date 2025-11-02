# Anthropic Engineering Digest - 输出目录

本目录存放从 Anthropic Engineering 博客生成的中文技术文档。

## 文件命名规则

所有文档采用统一命名格式：**`YYYYMMDD-topic.html`**

### 格式说明

- **日期部分**：`YYYYMMDD` 格式，表示文档生成日期
- **Topic 部分**：文章主题的英文简写，使用连字符 `-` 连接单词
- **文件扩展名**：`.html`

### 示例

```
20251028-Agent-Skills.html              # Agent Skills 文章
20251028-Claude-Code-Sandboxing.html    # Claude Code 沙箱机制
20251028-Prompt-Caching.html            # Prompt 缓存技术
```

## 文档特性

每份生成的 HTML 文档包含：

✅ **完整的中文翻译** - 保持技术术语准确性
✅ **盘古之白排版** - 中英文间自动添加空格
✅ **本地存储图片** - 自动下载到 `images/` 目录，避免外链失效
✅ **高质量配图** - 来自 Anthropic 官方，清晰美观
✅ **代码高亮** - 支持多种编程语言
✅ **响应式设计** - 自适应桌面和移动设备
✅ **专业排版** - Anthropic 官方风格
✅ **署名信息** - 整理者：Panda

## 目录结构

```
output/
├── README.md                      # 本说明文档
├── images/                        # 图片存储目录
│   ├── 973938b77fd2.png          # 文章配图（自动下载）
│   └── 75c57d8cca38.png          # 文章配图（自动下载）
├── 20251028-Agent-Skills.html    # 生成的文档
└── 20251028-Claude-Agent-SDK.html # 生成的文档
```

## 查看文档

直接在浏览器中打开 `.html` 文件即可查看。

## 生成工具

文档由 **Anthropic Digest Generator** 自动生成。

- 源代码：`../generator.py`
- 模板文件：`../template.html`
- Skill 定义：`../skill.md`
