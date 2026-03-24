---
title: "为什么很多 AI 公司不用 LangChain？为什么 90% 的 AI 应用不需要 Agent"
categories: [AI]
tags: [notes]
---

最近很多 AI 工程师会发现一个有点反直觉的现象：

* 很多公司 **从 LangChain 迁移到自研 Agent runtime**
* 同时 **大量 AI 产品其实根本没有使用 Agent**

这篇文章解释两个问题：

1. 为什么很多公司不用 LangChain
2. 为什么 90% 的 AI 应用其实不需要 Agent

---

# 一、为什么很多 AI 公司不用 LangChain

LangChain 并不是不好，它依然是 **最流行的 AI 框架**。
但在生产环境中，很多团队最终会选择 **自己实现 Agent runtime**。

主要原因有 5 个。

---

## 1 抽象层太厚，调试困难

LangChain 的核心问题之一是 **抽象层叠加过多**。

一个简单任务在 LangChain 里的结构通常像这样：

```
Agent
 └── Chain
      └── PromptTemplate
           └── LLM
                └── OutputParser
```

当结果不正确时，你很难判断问题在哪一层：

可能是：

* Prompt 写错
* Parser 解析错误
* Tool Schema 不匹配
* Agent 决策错误

很多团队最后发现：

> Debug LangChain 比直接写代码还复杂。

于是很多团队选择直接实现简单架构：

```
LLM API
+ Tool Router
+ State Machine
```

逻辑更加透明。

---

## 2 Agent Workflow 实际上是“状态机”

现实中的 Agent 很少是简单流程：

```
prompt → tool → answer
```

更常见的是：

```
plan
  ↓
search
  ↓
read
  ↓
summarize
  ↓
decide next step
```

本质上这是 **状态机 / DAG workflow**。

很多团队最后都会实现类似：

```
State {
  Planning
  Searching
  Reading
  Acting
  Finished
}
```

而 LangChain 的 Agent 是 **Prompt Loop 模式**：

```
while not finished:
    ask LLM what to do
```

这种方式的问题：

* 不可控
* Token 消耗大
* 行为难以限制

因此很多团队更倾向：

**显式状态机 + LLM辅助**

---

## 3 Token 成本问题

LangChain Agent 的典型行为：

每一步都会发送完整上下文：

```
User question
Tool result
Reasoning
History
Tools schema
```

假设一个：

```
10 step agent
```

可能消耗：

```
50k tokens
```

自研 runtime 可以优化：

* 压缩历史
* 分阶段 context
* 不重复发送 tool schema

很多团队能把成本 **降低 70% 以上**。

---

## 4 LangChain 的历史设计问题

LangChain 诞生于 **GPT-3 时代（2022）**。

当时 LLM API 非常简单，没有：

* Tool Calling
* Structured Output
* JSON Mode

因此 LangChain 发明了很多 workaround：

* OutputParser
* Agent Executor
* ReAct Prompt

但现在模型已经原生支持：

```
tool calling
json mode
structured output
```

很多 LangChain 组件显得有些 **过度设计**。

例如：

过去：

```
prompt + regex parser
```

现在：

```
model.json_schema()
```

---

## 5 性能问题

LangChain Python 常见问题：

* 多层 wrapper
* runtime reflection
* 动态 schema
* callback tracing

在高并发服务中：

```
1000 agent requests
```

CPU 消耗可能非常高。

很多 AI infra 团队最后会：

```
重写 runtime
```

例如：

| 公司         | 做法                    |
| ---------- | --------------------- |
| OpenAI     | 自研 Agents SDK         |
| Anthropic  | 自研 tool runtime       |
| Perplexity | 自研 agent orchestrator |
| Cursor     | 自研 agent runtime      |
| Notion AI  | 自研 workflow           |

---

## 现在很多 AI 产品的真实架构

很多 AI 产品的架构其实是：

```
LLM API
   │
Agent Runtime
   │
State Machine
   │
Tool Router
   │
External Systems
```

而不是：

```
LangChain
   │
Agent
   │
Chain
   │
LLM
```

---

## 为什么 Rig 这类框架开始流行

Rig 的理念是：

**LLM Runtime Library，而不是 AI Framework**

它只提供：

```
model
tool
embedding
agent
```

而这些部分需要开发者自己实现：

```
workflow
state machine
orchestration
```

这种模式其实更接近很多成熟团队的做法。

---

# 二、为什么 90% 的 AI 应用不需要 Agent

很多人刚接触 AI 时会觉得：

> 做 AI 应用必须要 Agent。

但现实是：

**大多数 AI 产品根本不需要 Agent。**

原因主要有 4 个。

---

## 1 大多数 AI 任务其实是“单步任务”

Agent 是为 **多步骤自主决策任务**设计的。

典型 Agent：

```
思考 → 选择工具 → 执行 → 再思考 → 再调用工具
```

但现实中的 AI 产品大多数只是：

```
用户输入 → LLM → 输出
```

例如：

| 产品功能    | 是否需要 Agent |
| ------- | ---------- |
| AI写邮件   | ❌          |
| AI写代码补全 | ❌          |
| AI总结文档  | ❌          |
| AI翻译    | ❌          |
| AI客服    | ❌          |

这些通常只需要：

```
prompt + LLM
```

最多再加一个：

```
RAG
```

---

## 2 Agent 会增加不确定性

Agent 本质是：

```
让 LLM 决定下一步做什么
```

但 LLM 不是确定性程序。

可能出现：

* 调错工具
* 无限循环
* 奇怪决策
* Token 爆炸

例如：

用户：

```
帮我查天气
```

Agent 可能走：

```
思考 → 搜索 → 再搜索 → 再总结 → 再思考
```

但工程师其实只需要：

```
getWeather(city)
```

因此很多团队更喜欢：

**确定 workflow，而不是 LLM 决策。**

---

## 3 Token 成本指数增长

Agent 每一步通常都会发送：

```
历史上下文
reasoning
tool schema
```

假设：

```
5 step agent
```

Token 消耗可能变成：

```
5x ~ 10x
```

很多公司最后发现：

```
Agent成本 > 模型成本
```

于是改成：

**固定流程 + LLM辅助**

---

## 4 很多所谓“Agent”其实只是 Workflow

很多 AI 产品宣传：

> 我们用了 AI Agent

但实际代码往往是：

```
Step1: 搜索
Step2: 阅读
Step3: 总结
Step4: 输出
```

本质是：

```
Workflow
```

而不是：

```
LLM 自主决策
```

区别在于：

### Agent

```
LLM 决定下一步
```

### Workflow

```
程序决定下一步
```

绝大多数公司选择：

**Workflow。**

---

# 三、真实行业比例（经验值）

很多 AI Infra 工程师的经验统计：

| 类型         | 比例  |
| ---------- | --- |
| 简单 LLM API | 60% |
| RAG        | 30% |
| Workflow   | 9%  |
| 真正 Agent   | <1% |

因此：

> 真正需要 Agent 的场景非常少。

---

# 四、真正适合 Agent 的场景

Agent 更适合 **开放任务**。

例如：

```
帮我研究 Tesla 最近的财报并给投资建议
```

需要：

```
搜索
阅读
分析
总结
```

步骤不固定。

或者：

```
自动处理客服邮件
```

涉及：

```
读取邮件
分类
调用CRM
自动回复
```

---

# 五、为什么 Agent Demo 很多

因为 Agent Demo **非常炫酷**。

例如：

```
AutoGPT
BabyAGI
CrewAI
```

看起来像：

```
AI自己完成复杂任务
```

但现实问题：

* 不稳定
* 很慢
* 很贵

所以很多公司最后做的是：

```
LLM + workflow engine
```

---

# 六、一个有趣的事实

很多成功的 AI 产品：

* ChatGPT
* Cursor
* Notion AI
* Perplexity

核心其实都不是 Agent。

而是：

```
LLM
+ RAG
+ Tool Calling
+ Workflow
```

而不是：

```
Fully autonomous Agent
```

---

# 总结

LangChain 和 Agent 依然有价值，但很多 AI 工程团队逐渐发现：

* LangChain 更适合 **快速原型**
* Agent 更适合 **开放复杂任务**

而绝大多数 AI 产品最终采用的架构是：

```
LLM
+ RAG
+ Workflow
+ Tool Calling
```

而不是：

```
全自动 Agent
```
