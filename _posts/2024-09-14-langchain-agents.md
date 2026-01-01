---
layout:       post
title:        "05. Agents 实战（中级）：规划与可控性的平衡"
author:       "iehmltym（張）"
header-style: text
catalog:      true
tags:
    - LangChain
    - Agents
    - 中级
---

面向读者：**中级**。场景：**生产级工单处理 Agent**。

## 背景/问题
Agent 能自己规划步骤，但容易发散、成本不可控。需要限制步数、限定工具。

## 核心概念
- Agent Executor
- 工具集合
- 迭代次数限制

## 方案设计
- 只提供必要工具
- 限制 `max_iterations`
- 输出必须结构化

## 关键实现（含代码）
**示例 1：最小 Agent（可运行）**

```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain.agents import create_react_agent, AgentExecutor
from langchain_core.prompts import ChatPromptTemplate

@tool
def fetch_ticket(ticket_id: str) -> dict:
    return {"ticket_id": ticket_id, "issue": "无法登录"}

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.2)
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是工单助手，谨慎调用工具。"),
    ("human", "处理工单：{input}")
])
agent = create_react_agent(llm, [fetch_ticket], prompt)
executor = AgentExecutor(agent=agent, tools=[fetch_ticket], max_iterations=3)

print(executor.invoke({"input": "T1001"}))
```

**意图与边界**：验证 Agent 执行；边界是缺少权限校验。

**示例 2：结构化收敛（可运行）**

```python
from pydantic import BaseModel
from langchain_core.output_parsers import JsonOutputParser
from langchain_core.prompts import PromptTemplate
from langchain_openai import ChatOpenAI

class TicketResult(BaseModel):
    summary: str
    action: str

parser = JsonOutputParser(pydantic_object=TicketResult)
prompt = PromptTemplate.from_template(
    "输出JSON：{format_instructions}\n工单描述：{text}"
).partial(format_instructions=parser.get_format_instructions())

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.2)
chain = prompt | llm | parser

print(chain.invoke({"text": "用户无法登录，需要重置密码"}))
```

**意图与边界**：让输出可落地；边界是模型可能拒答或格式错误。

## 常见坑与排错
- 工具数量过多导致 Agent 选择混乱。
- 缺少迭代限制导致长循环。

## 性能/安全考虑
- 将 Agent 只用于“探索型任务”。
- 对每次工具调用做审计日志。

## 测试与验证
- 固定输入跑回归测试，监控结果一致性。
- 超时场景要能降级到人工。

## 最小可复现示例
1. 配置 `OPENAI_API_KEY`。
2. 运行示例 1。
3. 预期输出：包含工单信息与处理建议。


## 进阶实践：生产级 Agent 的落地细节

### 1. 场景拆解与职责边界
- **输入边界**：明确用户输入范围（如客服工单/订单查询/知识问答），超出范围要有拒答与引导。
- **工具边界**：每个工具只做一件事（查询/写入/校验），避免工具“万能化”。
- **模型边界**：高风险问题强制走更稳定的模型或人工兜底。
- **数据边界**：需要处理 PII 的字段必须最小化使用与日志脱敏。

### 2. 关键链路的可观测性
- **请求 ID**：每一次请求都带 trace_id，覆盖检索、工具、模型调用。
- **Prompt 版本号**：输出日志中必须记录 prompt 版本和模型版本。
- **召回证据**：RAG 场景必须记录检索片段（可做 hash）。
- **错误分层**：区分“模型错误/工具错误/数据错误”。

### 3. 质量与稳定性指标（建议落地）
- **准确率**：Top-1 命中率、引用正确率。
- **忠实度**：回答是否严格来自检索资料。
- **拒答率**：超出能力范围的问题是否正确拒答。
- **一致性**：同样输入多次输出的稳定程度。
- **成本**：每千次请求的 token 成本与工具调用成本。

### 4. 典型业务流程模板（可直接复用）
1. **输入校验** → 2. **意图分类** → 3. **路由模型** → 4. **检索/工具调用** → 5. **结构化输出** → 6. **安全审计** → 7. **回写系统/响应用户**。

### 5. 生产检查清单（节选）
- [ ] 是否启用请求级 trace_id
- [ ] Prompt 与模型版本是否在日志中可追溯
- [ ] 工具调用是否有超时与重试
- [ ] 是否配置回退策略（模型/人工）
- [ ] 是否做过最小红队测试（越权/敏感信息）

### 6. 可复用的日志结构（示例）
```json
{
  "trace_id": "req_123",
  "prompt_version": "v3",
  "model": "gpt-4o-mini",
  "retrieval_ids": ["doc_12", "doc_57"],
  "tool_calls": ["get_order_status"],
  "latency_ms": 820,
  "result": "success"
}
```

### 7. 故障演练清单
- 模型超时
- 工具返回异常
- 检索为空
- 输出格式解析失败
- 不可用依赖（向量库/缓存）

### 8. 成本与容量估算（简化模型）
- 日请求量：1万
- 平均每次调用：1.5k tokens
- 预计日成本 = 日请求量 × 每次 token 费用
- 对热点问题做缓存可降低 30%-60% 成本

## 扩展复现步骤（面向生产级 Agent）
1. 准备一份 **最小 FAQ 文档**（至少 10 条）。
2. 构建向量库并写入 metadata（source、版本号）。
3. 配置路由策略（easy/hard）与模型列表。
4. 启动一个最小 API（例如 FastAPI）对外提供 `/ask`。
5. 用 20 个问题跑回归测试并记录指标。

**预期输出**：
- 对标准问题返回稳定答案
- 对超出范围问题正确拒答
- 日志中可追踪 prompt 版本与检索证据


## 总结
生产级 Agent 不是越自由越好，而是**可控+可审计**。
