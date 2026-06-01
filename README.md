<div align="center">

# 📚 AI Digest · AI 内容速读

**用 AI 速读 AI** — 精选并翻译 AI 前沿工程与研究内容的中文知识库。
*A curated Chinese digest of frontier AI engineering & research.*

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)
[![Pages](https://img.shields.io/badge/site-GitHub%20Pages-brightgreen.svg)](https://myagenthubs.github.io/ai-digest/)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-orange.svg)](#-贡献--contributing)

### 🌐 在线阅读 · Read online

**[👉 中文站点 / Live site](https://myagenthubs.github.io/ai-digest/)**　|　**[📦 GitHub 仓库 / Repo](https://github.com/MyAgentHubs/ai-digest)**

### 🗂 语言 · Language

**[简体中文](#中文)**　|　**[English](#english)**

</div>

---

## 中文

### 这是什么

**AI Digest** 是一个开放的中文 AI 内容速读站。我们把 Anthropic、OpenAI 等机构的前沿**工程博客**与**研究论文**翻译、提炼成易读的中文索引，并配以自托管 AI 网关 **OpenClaw** 的实战手册。每篇条目都附上**原文链接**，方便对照阅读。

站点由静态 HTML 构成，可直接在 [GitHub Pages](https://myagenthubs.github.io/ai-digest/) 浏览。

### 📖 内容板块

| 板块 | 说明 | 入口 |
|------|------|------|
| 🦞 **OpenClaw 实战手册** | 自托管 AI 网关的部署、排查、安全与版本演进 | [打开 →](practice/openclaw/index.html) |
| 🏢 **Anthropic Engineering** | Anthropic 官方工程博客（中文翻译索引） | [打开 →](engineering/anthropic/index.html) |
| 🔬 **Anthropic Research** | Anthropic 官方研究论文（中文翻译索引） | [打开 →](research/anthropic/index.html) |
| 🤖 **OpenAI Engineering** | OpenAI 产品 / 平台 / 工程文章（中文翻译索引） | [打开 →](engineering/openai/index.html) |
| 🧪 **OpenAI Research** | OpenAI 研究与安全文章（中文翻译索引） | [打开 →](research/openai/index.html) |

### 🗂 目录结构

```
ai-digest/
├── index.html              # 站点首页（汇总各板块）
├── engineering/
│   ├── anthropic/          # Anthropic 工程博客索引
│   └── openai/             # OpenAI 工程 / 平台索引
├── research/
│   ├── anthropic/          # Anthropic 研究索引
│   └── openai/             # OpenAI 研究索引
└── practice/
    └── openclaw/           # OpenClaw 自托管网关实战手册
```

### 🚀 本地预览

```bash
git clone https://github.com/MyAgentHubs/ai-digest.git
cd ai-digest
# 任选其一启动本地静态服务器
python3 -m http.server 8000
# 然后打开 http://localhost:8000
```

### 🤝 贡献 · Contributing

欢迎补充翻译、勘误或新增板块：

1. Fork 仓库并新建分支。
2. 在对应板块目录下新增内容，**每条必须附上原文 URL**（保持 `查看原文 →` 链接）。
3. 沿用现有索引页样式；新增文章请同步更新该板块的 `index.html`，并在根 `index.html` 挂上入口。
4. 内容须可追溯、标注来源与日期，不得编造（参见 [`practice/openclaw/MAINTENANCE.md`](practice/openclaw/MAINTENANCE.md) 的内容规范）。
5. 提交 Pull Request，说明改动内容与来源。

### 📄 许可

内容采用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 许可，可在署名前提下自由分享与改编。原文版权归各自机构所有，本站仅作中文整理与索引。

---

## English

### What is this

**AI Digest** is an open, Chinese-language reading hub for frontier AI. It translates and distills the **engineering blogs** and **research papers** of organizations like Anthropic and OpenAI into easy-to-scan Chinese indexes, alongside a hands-on handbook for the self-hosted AI gateway **OpenClaw**. Every entry links back to its **original source**.

The site is plain static HTML, served via [GitHub Pages](https://myagenthubs.github.io/ai-digest/).

### 📖 Sections

| Section | About | Open |
|---------|-------|------|
| 🦞 **OpenClaw Handbook** | Deploy, troubleshoot, secure & track the self-hosted AI gateway | [Open →](practice/openclaw/index.html) |
| 🏢 **Anthropic Engineering** | Anthropic's engineering blog (Chinese index) | [Open →](engineering/anthropic/index.html) |
| 🔬 **Anthropic Research** | Anthropic's research papers (Chinese index) | [Open →](research/anthropic/index.html) |
| 🤖 **OpenAI Engineering** | OpenAI product / platform / engineering posts (Chinese index) | [Open →](engineering/openai/index.html) |
| 🧪 **OpenAI Research** | OpenAI research & safety posts (Chinese index) | [Open →](research/openai/index.html) |

### 🗂 Layout

```
ai-digest/
├── index.html              # Home page (all sections)
├── engineering/{anthropic,openai}/
├── research/{anthropic,openai}/
└── practice/openclaw/      # OpenClaw self-hosted gateway handbook
```

### 🚀 Local preview

```bash
git clone https://github.com/MyAgentHubs/ai-digest.git
cd ai-digest && python3 -m http.server 8000   # open http://localhost:8000
```

### 🤝 Contributing

Translations, fixes, and new sections are welcome:

1. Fork and branch.
2. Add content under the right section — **every entry must carry its original source URL** (keep the `查看原文 →` link).
3. Follow the existing index-page style; update the section's `index.html` and link it from the root `index.html`.
4. Keep content traceable and dated; don't fabricate (see [`practice/openclaw/MAINTENANCE.md`](practice/openclaw/MAINTENANCE.md)).
5. Open a PR explaining the change and its sources.

### 📄 License

Content is under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — share and adapt with attribution. Original works remain the property of their respective organizations; this site only provides Chinese curation and indexing.

---

<div align="center">
<sub>© 2026 ai-digest · 用 AI 速读 AI · 整理者：Panda</sub>
</div>
