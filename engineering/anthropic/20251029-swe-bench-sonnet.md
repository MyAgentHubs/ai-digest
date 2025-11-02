# 用 Claude 3.5 Sonnet 提升 SWE-bench Verified 的表现

![SWE-bench 性能示意图](https://www-cdn.anthropic.com/images/4zrzovbb/website/290e18c396589364580e2432e96d7d96724338a7-2554x2554.svg)

发布于 2025 年 1 月 6 日

## 概述

升级后的 Claude 3.5 Sonnet 在 SWE-bench Verified 上达到了 49% 的成绩，超越了之前 45% 的最先进水平。本文介绍了围绕该模型构建的"智能体"系统，旨在帮助开发者充分发挥 Claude 3.5 Sonnet 的性能潜力。

## SWE-bench 简介

SWE-bench（软件工程基准）是一项 AI 评估基准，用于评估模型完成真实软件工程任务的能力。具体而言，它测试模型解决流行开源 Python 仓库中 GitHub issue 的能力。

在基准中的每个任务中，AI 模型获得一个配置好的 Python 环境和 issue 解决前的仓库副本。模型需要理解、修改和测试代码，最后提交解决方案。

每个解决方案都会与关闭原始 GitHub issue 的拉取请求中的实际单元测试进行对比。这验证了 AI 模型是否能实现与原始 PR 作者相同的功能。

## 为什么选择 SWE-bench？

SWE-bench 因以下几个原因获得了广泛关注：

1. **真实工程任务** - 采用实际项目中的任务，而非竞赛或面试风格的问题
2. **未饱和空间** - 改进空间充足。至本文发稿时，尚无模型突破 SWE-bench Verified 的 50% 大关（Claude 3.5 Sonnet 新版本为 49%）
3. **整体系统评估** - 测试完整的"智能体"而非孤立的模型。开源开发者和初创公司在优化框架方面取得了显著成果

## 基准说明

原始 SWE-bench 数据集包含一些无法仅从 GitHub issue 解决的任务。SWE-bench Verified 是由人工审核的 500 题子集，确保任务均可解决，因此提供了最清晰的智能体性能衡量。

## 技术架构

### 设计理念

在为升级的 Claude 3.5 Sonnet 优化智能体框架时，我们秉持的哲学是：**赋予语言模型尽可能多的控制权，保持框架最小化**。

智能体包含三个核心组件：
- 初始提示词
- Bash 工具（执行 bash 命令）
- 编辑工具（查看和编辑文件）

系统持续采样，直到模型自行决定完成或达到 200k token 上下文限制。

### 提示词策略

提示词概述了建议的方法，但不过度冗长。模型可自由选择推进方式，无需遵循严格的离散步骤。当不考虑 token 成本时，鼓励模型生成较长的响应会有帮助。

**示例提示词结构：**

```plaintext
<uploaded_files>
{location}
</uploaded_files>
我已将 Python 代码仓库上传到 {location} 目录。请考虑以下 PR 描述：

<pr_description>
{pr_description}
</pr_description>

能否帮我实现必要的改动以满足 PR 描述中的要求？
测试文件的所有改动已由我处理。你无需修改测试逻辑！

按以下步骤解决问题：
1. 首先探索仓库结构
2. 编写脚本复现错误
3. 修改源代码
4. 重新运行脚本验证修复
5. 考虑边界情况
```

### Bash 工具规范

```json
{
   "name": "bash",
   "description": "在 bash shell 中运行命令\n* 此工具不需要 XML 转义\n* 无互联网访问\n* 可通过 apt 和 pip 访问常见的 Linux 和 Python 包\n* 状态在命令调用间持久化\n* 查看特定行范围使用：sed -n 10,25p /path/to/file\n* 避免产生大量输出\n* 长运行命令请后台执行",
   "input_schema": {
       "type": "object",
       "properties": {
           "command": {
               "type": "string",
               "description": "要运行的 bash 命令"
           }
       },
       "required": ["command"]
   }
}
```

### 编辑工具设计

编辑工具更为复杂，提供文件查看、创建和编辑的完整功能。工具描述包含详细的使用指导。

我们在工具描述和规范上投入大量精力，在多种智能体任务中进行了测试，以预见模型可能的误解。

**关键特性：**
- 状态在命令调用间持久化
- `view` 命令对文件返回带行号的输出，对目录列出 2 层深度的文件
- `create` 命令不能用于已存在的文件
- `str_replace` 命令要求 `old_str` 在文件中有唯一匹配

**工具规范片段：**

```json
{
   "name": "str_replace_editor",
   "description": "文件查看、创建和编辑工具...",
   "input_schema": {
       "type": "object",
       "properties": {
           "command": {
               "type": "string",
               "enum": ["view", "create", "str_replace", "insert", "undo_edit"],
               "description": "允许的操作命令"
           },
           "path": {
               "type": "string",
               "description": "绝对路径，如 /repo/file.py 或 /repo"
           }
       },
       "required": ["command", "path"]
   }
}
```

### 容错设计

我们通过多种方式改进了工具的容错能力。例如，模型在离开根目录后可能会混淆相对路径，所以我们要求所有路径都必须是绝对路径。

对于文件编辑，我们尝试了多种策略，发现字符串替换最为可靠：模型指定 `old_str` 被替换为 `new_str`。仅当有唯一匹配时替换才会执行。

## 性能结果

升级的 Claude 3.5 Sonnet 展现了比以前模型和现有最先进模型更强的推理、编码和数学能力。工具和框架优化帮助充分发挥了这些能力。

| 模型 | **Claude 3.5 Sonnet（新）** | 前最先进 | Claude 3.5 Sonnet（旧） | Claude 3 Opus |
|------|---------------------------|---------|----------------------|---------------|
| SWE-bench Verified 成绩 | **49%** | 45% | 33% | 22% |

## 实际案例分析

### 案例：RidgeClassifierCV 参数问题

我们使用 SWE-Agent 框架作为基础。在日志中，我们将智能体的文本输出、工具调用和响应分别表示为 THOUGHT（思考）、ACTION（行动）和 OBSERVATION（观察）。

**初始提示（已填入真实 PR 描述）：**

```plaintext
<uploaded_files>
/repo
</uploaded_files>
考虑以下 PR 描述：

<pr_description>
linear_model.RidgeClassifierCV 的 store_cv_values 参数问题

#### 问题描述
sklearn.linear_model.RidgeClassifierCV 的 store_cv_values 参数出错

#### 复现步骤
import numpy as np
from sklearn import linear_model as lm

n = 100
x = np.random.randn(n, 30)
y = np.random.normal(size = n)

rr = lm.RidgeClassifierCV(alphas = np.arange(0.1, 1000, 0.1),
                          normalize = True,
                          store_cv_values = True).fit(x, y)

#### 预期结果
获得通常的岭回归模型输出，并保留交叉验证预测作为属性

#### 实际结果
TypeError: __init__() got an unexpected keyword argument 'store_cv_values'
</pr_description>
```

**第一步 - 探索仓库：**

```plaintext
THOUGHT:
让我按照步骤实现必要的改动。首先探索仓库结构。

ACTION (str_replace_editor):
command: view
path: /repo

OBSERVATION:
/repo 中 2 层深度的文件和目录：
/repo
/repo/PULL_REQUEST_TEMPLATE.md
/repo/ISSUE_TEMPLATE.md
/repo/sklearn
/repo/sklearn/decomposition
...
```

**第二步 - 创建复现脚本：**

```plaintext
THOUGHT:
基于仓库结构，我需要修改 sklearn/linear_model 目录下的 Ridge 分类器。
让我创建一个脚本来复现错误：

ACTION (str_replace_editor):
command: create
path: /repo/reproduce_error.py
file_text:
import numpy as np
from sklearn import linear_model as lm

n = 100
x = np.random.randn(n, 30)
y = np.random.normal(size = n)

rr = lm.RidgeClassifierCV(alphas=np.arange(0.1, 1000, 0.1),
                          normalize=True,
                          store_cv_values=True).fit(x, y)

OBSERVATION:
文件已成功创建：/repo/reproduce_error.py
```

**第三步 - 验证错误：**

```plaintext
THOUGHT:
验证我们是否能复现错误：

ACTION (bash):
command: python3 /repo/reproduce_error.py

OBSERVATION:
Traceback (most recent call last):
  File "/repo/reproduce_error.py", line 10, in <module>
    store_cv_values=True).fit(x, y)
TypeError: __init__() got an unexpected keyword argument 'store_cv_values'
```

**第四步 - 修复代码：**

```plaintext
THOUGHT:
问题在于 RidgeClassifierCV 类虽然继承自 _BaseRidgeCV，但在构造函数中
没有将 store_cv_values 参数传递给父类。

让我修复这个问题：

ACTION (str_replace_editor):
command: str_replace
path: /repo/sklearn/linear_model/ridge.py

old_str:
    def __init__(self, alphas=(0.1, 1.0, 10.0), fit_intercept=True,
                 normalize=False, scoring=None, cv=None, class_weight=None):
        super(RidgeClassifierCV, self).__init__(
            alphas=alphas, fit_intercept=fit_intercept, normalize=normalize,
            scoring=scoring, cv=cv)
        self.class_weight = class_weight

new_str:
    def __init__(self, alphas=(0.1, 1.0, 10.0), fit_intercept=True,
                 normalize=False, scoring=None, cv=None, class_weight=None,
                 store_cv_values=False):
        super(RidgeClassifierCV, self).__init__(
            alphas=alphas, fit_intercept=fit_intercept, normalize=normalize,
            scoring=scoring, cv=cv, store_cv_values=store_cv_values)
        self.class_weight = class_weight
```

在这个特定案例中，模型经过 12 步后决定完成任务。任务测试随后成功运行，验证了解决方案的有效性。某些任务需要超过 100 轮的交互；在其他情况下，模型在上下文耗尽前一直尝试。

升级后的 Claude 3.5 Sonnet 相比旧模型更频繁地自我纠正，并展现了尝试多个解决方案的能力，而非陷入重复错误。

## 遇到的挑战

SWE-bench Verified 是强大的评估，但比简单的单轮评估更复杂。我们遇到的挑战包括：

### 1. 时长和高 Token 成本

上述案例仅用 12 步完成。然而，许多成功运行需要数百轮交互，消耗超 100k tokens。升级的 Claude 3.5 Sonnet 具有较强的持久性：给予充足时间通常能解决问题，但成本可能高昂。

### 2. 评分问题

检查失败任务时，我们发现模型表现正确但存在环境设置问题，或者安装补丁被应用两次。解决这些系统问题对准确衡量智能体性能至关重要。

### 3. 隐藏测试

由于模型看不到评分用的测试，它常会误认为成功。某些失败源于模型在错误的抽象层解决问题（打补丁而非深层重构）。有些失败则更有争议：问题被解决但不符合原始任务的单元测试。

### 4. 多模态支持

尽管升级的 Claude 3.5 Sonnet 具备出色的视觉和多模态能力，我们未实现让它查看文件系统中保存的文件或引用的 URL。这使调试某些任务（尤其是 Matplotlib 相关任务）特别困难，容易导致模型幻觉。这里有明显的改进空间。SWE-bench 已推出专注于多模态任务的新评估，我们期待看到开发者利用 Claude 在该评估上取得更高成绩。

## 总结

升级的 Claude 3.5 Sonnet 以简洁的提示和两个通用工具在 SWE-bench Verified 上取得 49% 的成绩，超越了现有最先进的 45%。我们相信基于新 Claude 3.5 Sonnet 构建的开发者将迅速找到比我们初步展示的方法更优的方式来改进 SWE-bench 成绩。

## 致谢

Erik Schluntz 优化了 SWE-bench 智能体并撰写了本文。Simon Biggs、Dawn Drain 和 Eric Christiansen 协助实现了基准。Shauna Kravec、Dawn Drain、Felipe Rosso、Nova DasSarma、Ven Chandrasekaran 及许多其他人为培训 Claude 3.5 Sonnet 的智能体编码能力做出了贡献。
