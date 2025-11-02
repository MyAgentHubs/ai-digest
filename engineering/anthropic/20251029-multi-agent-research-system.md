# 我们如何构建多 Agent 研究系统

**原文链接**: https://www.anthropic.com/engineering/multi-agent-research-system
**发布日期**: 2025 年 6 月 13 日

Claude 现在具备了研究能力（Research capabilities），可以在网络、Google Workspace 以及任何集成服务中进行搜索，以完成复杂任务。

这个多 Agent 系统从原型到生产的旅程，让我们学到了关于系统架构、工具设计和提示工程的关键经验。多 Agent 系统由多个专门的 Agent 组成，它们协同工作以完成复杂任务。

本文分解了对我们有效的原则——我们希望你在构建自己的多 Agent 系统时能发现这些原则很有用。

## 为什么研究需要多 Agent 系统

研究工作涉及开放性问题，很难提前预测所需的步骤。你无法为探索复杂主题硬编码固定路径，因为这个过程本质上是动态的。

这种不可预测性使得 AI Agent 特别适合研究任务。研究需要灵活性，能够在调查展开时转向或探索相关联系。模型必须能够：

- **动态规划**：根据新发现调整策略
- **深度探索**：追踪有希望的线索
- **综合信息**：从多个来源整合见解
- **迭代改进**：基于中间结果完善方法

单个 Agent 可以处理简单任务，但复杂研究需要协调。这就是多 Agent 系统的用武之地。

## 我们的多 Agent 架构

我们的研究系统采用分层架构，具有明确的职责分离：

### 1. Orchestrator（编排器）

**职责**：高层规划和任务分配

Orchestrator 是系统的大脑。它：

- 分解研究问题为子任务
- 决定哪些 Agent 应该处理每个任务
- 监控进度并调整计划
- 综合最终结果

**关键设计决策**：

- 使用最强大的模型（Claude 3.7 Sonnet）以获得最佳推理能力
- 保持 Orchestrator 的提示简洁且专注于规划
- 避免让 Orchestrator 直接执行任务——它应该委派

### 2. Research Agent（研究 Agent）

**职责**：执行具体的研究任务

Research Agent 是系统的工作马。每个 Agent：

- 接收来自 Orchestrator 的明确定义的任务
- 使用专门的工具（搜索、文档检索等）
- 收集和分析信息
- 向 Orchestrator 报告发现

**关键设计决策**：

- 每个 Research Agent 专注于单一任务
- 提供清晰的工具和期望
- 使用结构化输出格式以便于解析

### 3. Synthesis Agent（综合 Agent）

**职责**：整合和格式化结果

Synthesis Agent 将所有内容汇集在一起。它：

- 从多个 Research Agent 收集结果
- 识别模式和关键见解
- 创建连贯的最终报告
- 应用格式和风格指南

**关键设计决策**：

- 在综合前等待所有研究完成
- 使用模板确保一致的输出
- 优先考虑清晰度和可操作性

## 我们学到的核心原则

### 原则 1：明确的职责分离

**问题**：早期版本中，Agent 试图同时规划、搜索和综合。这导致了混乱的提示和不可预测的行为。

**解决方案**：严格分离关注点。每个 Agent 应该有一个主要职责。

**实践**：

```markdown
# Orchestrator 提示
你的唯一职责是规划和委派。
- 不要执行搜索
- 不要编写最终报告
- 只创建任务并分配它们

# Research Agent 提示
你的唯一职责是收集信息。
- 使用提供的工具进行搜索
- 总结你的发现
- 不要试图规划整体策略
```

### 原则 2：结构化通信

**问题**：Agent 之间的自由文本通信导致了解析错误和误解。

**解决方案**：使用严格的结构化格式进行 Agent 间通信。

**实践**：

```json
{
  "task_id": "research_001",
  "type": "web_search",
  "parameters": {
    "query": "Claude 3.7 capabilities",
    "depth": "detailed"
  },
  "status": "completed",
  "results": {
    "summary": "...",
    "sources": [...]
  }
}
```

### 原则 3：显式工具约束

**问题**：给 Agent 太多工具会导致决策瘫痪和低效使用。

**解决方案**：为每个 Agent 类型提供最小且相关的工具集。

**实践**：

- **Orchestrator**：只有任务管理工具
- **Research Agent**：搜索 + 文档检索
- **Synthesis Agent**：格式化和模板工具

### 原则 4：渐进式复杂度

**问题**：试图一次构建完整系统导致了调试困难。

**解决方案**：从简单开始，逐步添加复杂性。

**我们的路径**：

1. **阶段 1**：单个 Research Agent 带工具
2. **阶段 2**：添加 Orchestrator 来管理多个 Research Agent
3. **阶段 3**：添加 Synthesis Agent 用于报告生成
4. **阶段 4**：添加专门的 Agent 用于特定任务

### 原则 5：主动错误处理

**问题**：Agent 失败会默默地破坏整个研究流程。

**解决方案**：在每个层级实现显式错误处理和重试逻辑。

**实践**：

```python
try:
    result = research_agent.execute(task)
except ToolError as e:
    # 使用不同参数重试
    result = research_agent.execute(task, fallback=True)
except AgentError as e:
    # 通知 Orchestrator 重新规划
    orchestrator.handle_failure(task, error=e)
```

### 原则 6：可观测性至关重要

**问题**：难以理解多 Agent 系统内部发生了什么。

**解决方案**：在每个层级构建全面的日志和监控。

**我们跟踪的内容**：

- Agent 决策和理由
- 工具调用和结果
- 任务完成时间
- 跨 Agent 的依赖关系

### 原则 7：人机协作而非自动化

**问题**：完全自主的系统做出了错误的假设。

**解决方案**：在关键决策点设计人类参与。

**实践**：

- 在开始深度研究前显示研究计划
- 允许用户引导方向
- 提供中间结果以便反馈

## 实际实现细节

### Orchestrator 提示结构

```markdown
你是研究 Orchestrator。你的职责是：

1. 分析用户的研究问题
2. 将其分解为具体、可操作的子任务
3. 将任务分配给专门的 Research Agent
4. 监控进度并根据需要调整

输出格式：
{
  "plan": [
    {"task_id": "...", "type": "...", "description": "..."},
    ...
  ]
}

约束：
- 每个任务必须是独立的
- 使用明确、具体的语言
- 不要执行研究——只规划
```

### Research Agent 提示结构

```markdown
你是专门的 Research Agent。给定一个任务，你必须：

1. 使用提供的工具收集相关信息
2. 分析和总结你的发现
3. 以结构化格式报告结果

可用工具：
- web_search(query: str)
- retrieve_document(doc_id: str)

输出格式：
{
  "task_id": "...",
  "findings": "...",
  "sources": [...],
  "confidence": "high|medium|low"
}
```

### 工具设计最佳实践

**1. 幂等性**

工具应该在相同输入下产生一致的结果：

```python
@tool
def search_web(query: str, max_results: int = 5):
    """搜索网络并返回排名靠前的结果"""
    # 确保确定性排序
    results = perform_search(query)
    return sorted(results, key=lambda x: x.score)[:max_results]
```

**2. 清晰的错误消息**

```python
if not query:
    raise ToolError(
        "search_web 需要非空查询。"
        "示例：search_web('Claude 3.7 features')"
    )
```

**3. 渐进式信息披露**

```python
@tool
def search_web(
    query: str,
    depth: Literal["quick", "standard", "comprehensive"] = "standard"
):
    """
    在网络上搜索。

    - quick: 前 3 个结果
    - standard: 前 10 个结果，带摘要
    - comprehensive: 前 20 个结果，带详细分析
    """
```

## 性能优化

### 1. 并行化

独立任务应该并行运行：

```python
# 不好：顺序执行
results = []
for task in tasks:
    result = agent.execute(task)
    results.append(result)

# 好：并行执行
results = await asyncio.gather(*[
    agent.execute(task) for task in tasks
])
```

### 2. 结果缓存

重复查询应该使用缓存：

```python
@cache(ttl=3600)  # 缓存 1 小时
def search_web(query: str):
    return expensive_search_operation(query)
```

### 3. 智能批处理

将相关请求分组：

```python
# 批量搜索而不是单独搜索
results = search_batch([
    "Claude 3.7 features",
    "Claude pricing",
    "Claude API limits"
])
```

## 评估和迭代

### 我们如何衡量成功

**1. 任务完成率**

- 系统成功完成了多少百分比的研究任务？
- 目标：>90%

**2. 结果质量**

- 人类评估者对输出评分（1-5 分）
- 目标：平均 >4.0

**3. 效率**

- 完成时间与预期的对比
- Agent 调用次数
- 目标：<5 分钟用于标准研究

**4. 成本**

- 每个研究任务的 token 使用量
- 目标：<100K tokens 用于标准任务

### 持续改进

我们每周运行评估流程：

1. **收集失败案例**：系统未能完成任务的地方
2. **分析根本原因**：提示问题？工具限制？架构缺陷？
3. **实施修复**：更新提示、添加工具或重构架构
4. **重新测试**：验证改进有效

## 生产经验教训

### 1. 提示漂移是真实存在的

**问题**：随着时间推移，我们不断调整提示，导致不一致的行为。

**解决方案**：

- 在版本控制中维护提示
- 运行回归测试以捕获意外变化
- 记录每次提示更改的原因

### 2. 工具可靠性至关重要

**问题**：不可靠的工具（超时、错误）破坏了整个流程。

**解决方案**：

- 实现重试和回退
- 监控工具性能
- 为关键功能提供替代工具

### 3. 上下文管理很棘手

**问题**：长时间运行的研究任务超出了上下文窗口。

**解决方案**：

- 在 Agent 之间主动总结和传递信息
- 使用外部存储用于大型中间结果
- 实现上下文修剪策略

### 4. 用户期望很重要

**问题**：用户不确定系统在做什么或需要多长时间。

**解决方案**：

- 显示实时进度更新
- 解释 Agent 决策
- 提供完成时间估计

## 未来方向

我们正在探索的内容：

### 1. 自适应 Agent 选择

根据任务特征动态选择最佳 Agent 配置。

### 2. 学习型 Orchestrator

Orchestrator 从过去的任务中学习以改进规划。

### 3. 人类反馈循环

在研究过程中集成更复杂的人类引导。

### 4. 跨研究记忆

在研究会话之间保留和重用知识。

## 开始构建

如果你想构建类似的系统，这是我们的建议：

### 从简单开始

```python
# 阶段 1：单个 Agent 带一个工具
agent = Agent(
    model="claude-3-7-sonnet",
    tools=[search_tool]
)

result = agent.execute("研究 Claude 的功能")
```

### 添加编排

```python
# 阶段 2：多个 Agent 带编排
orchestrator = Orchestrator(model="claude-3-7-sonnet")
research_agents = [Agent(tools=[search_tool]) for _ in range(3)]

tasks = orchestrator.plan("研究 AI 安全")
results = orchestrator.execute(tasks, agents=research_agents)
```

### 添加专门化

```python
# 阶段 3：专门的 Agent
web_agent = Agent(tools=[search_tool])
doc_agent = Agent(tools=[retrieve_tool])
synthesis_agent = Agent(tools=[format_tool])

# 自定义编排逻辑
```

## 关键要点

1. ✅ **明确职责**：每个 Agent 应该有一个清晰的单一目的
2. ✅ **结构化通信**：使用严格格式进行 Agent 间消息
3. ✅ **最小工具集**：只给每个 Agent 需要的工具
4. ✅ **渐进式构建**：从简单开始，逐步添加复杂性
5. ✅ **主动监控**：在每个层级实现全面的日志记录
6. ✅ **错误处理**：优雅地处理失败，带重试和回退
7. ✅ **人类参与**：在关键决策点设计协作

## 附录

### 完整示例代码

```python
from anthropic import Anthropic

client = Anthropic()

# Orchestrator
orchestrator_prompt = """
你是研究 Orchestrator。将研究问题分解为 2-3 个子任务。

输出 JSON：
{
  "tasks": [
    {"id": "1", "type": "search", "query": "..."},
    ...
  ]
}
"""

# Research Agent
research_prompt = """
使用 search_tool 收集信息。总结发现。

输出 JSON：
{
  "findings": "...",
  "sources": [...]
}
"""

# 执行研究
def run_research(question: str):
    # 规划阶段
    plan = client.messages.create(
        model="claude-3-7-sonnet",
        messages=[{
            "role": "user",
            "content": f"{orchestrator_prompt}\n\n问题：{question}"
        }]
    )

    # 执行阶段
    tasks = parse_json(plan.content)
    results = []

    for task in tasks:
        result = client.messages.create(
            model="claude-3-7-sonnet",
            messages=[{
                "role": "user",
                "content": f"{research_prompt}\n\n任务：{task}"
            }],
            tools=[search_tool]
        )
        results.append(result)

    return results
```

### 推荐资源

- [Anthropic API 文档](https://docs.anthropic.com)
- [工具使用指南](https://docs.anthropic.com/claude/docs/tool-use)
- [提示工程最佳实践](https://docs.anthropic.com/claude/docs/prompt-engineering)

---

**整理者**: Panda
**整理日期**: 2025 年 10 月 29 日
