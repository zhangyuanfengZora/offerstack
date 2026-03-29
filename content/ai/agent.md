---
title: Agent 框架
weight: 2
---

## 什么是 AI Agent？

**Agent** 是具有自主决策和行动能力的 AI 系统，能够通过调用工具完成复杂任务。

```
用户目标
    ↓
LLM 思考（Thought）
    ↓
选择并调用工具（Action）
    ↓
获取工具结果（Observation）
    ↓
继续思考... → 直到任务完成
    ↓
最终回答（Final Answer）
```

## Q: ReAct 框架是什么？

**ReAct = Reasoning + Acting**，交替进行推理和行动：

```
Thought: 我需要查询今天的天气
Action: weather_tool({"city": "北京"})
Observation: 北京今天晴天，25°C

Thought: 已获取天气信息，可以回答了
Final Answer: 北京今天天气晴朗，温度25°C，适合出行
```

## Q: Function Calling vs ReAct？

| 对比 | Function Calling | ReAct |
|------|-----------------|-------|
| 实现方式 | 模型原生支持 | Prompt 工程 |
| 可靠性 | 更高（结构化输出）| 较低（依赖 Prompt）|
| 适用模型 | GPT-4/Claude 等 | 所有 LLM |
| 工具定义 | JSON Schema | 文本描述 |

## Q: 多 Agent 协作模式？

```
主 Agent（Orchestrator）
    ├── 搜索 Agent    → 负责信息检索
    ├── 代码 Agent    → 负责编写执行代码
    ├── 分析 Agent    → 负责数据分析
    └── 写作 Agent    → 负责生成报告
```

常见框架：
- **LangGraph**：基于图的 Agent 工作流
- **AutoGen**：微软的多 Agent 对话框架
- **CrewAI**：角色扮演式多 Agent 协作
