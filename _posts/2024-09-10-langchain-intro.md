---
layout:       post
title:        "01. LangChain 全景入门（初学）：生产级 Agent 的第一步"
author:       "iehmltym（張）"
header-style: text
catalog:      true
tags:
    - LangChain
    - 入门
    - Agent
    - 生产实践
---

面向读者：**初学**。场景：**生产级客服 Agent**（多轮对话、工具调用、可审计）。

## 背景/问题
把大模型直接接到客服系统，常见问题是：回复不稳定、缺乏工具调用、难以审计。需要一个可组合、可测试的框架来组织提示词、工具、检索和日志。

## 核心概念
- **PromptTemplate**：参数化提示词。
- **LCEL**：用管道组合流程。
- **工具调用**：让 Agent 执行“动作”。
- **输出解析**：保证结构化结果。

## 方案设计
- 用 PromptTemplate 规范输入输出。
- 用 LCEL 把“问题 → 模型 → 结构化输出”串起来。
- 先不引入复杂 Agent，保证最小链路稳定。

## 关键实现（含代码）
**示例 1：最小链路（可运行）**

```python
from langchain_core.prompts import PromptTemplate
from langchain_openai import ChatOpenAI

prompt = PromptTemplate.from_template(
    "你是客服助手，请用三句话回答：{question}"
)
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.2)
chain = prompt | llm

result = chain.invoke({"question": "如何重置密码？"})
print(result.content)
```

**意图与边界**：快速验证模型可用性；边界是没有工具调用与审计。

**示例 2：结构化输出（可运行）**

```python
from langchain_core.output_parsers import JsonOutputParser
from pydantic import BaseModel, Field
from langchain_core.prompts import PromptTemplate
from langchain_openai import ChatOpenAI

class Answer(BaseModel):
    summary: str = Field(description="一句话总结")
    steps: list[str] = Field(description="操作步骤")

parser = JsonOutputParser(pydantic_object=Answer)

prompt = PromptTemplate.from_template(
    "请输出JSON，格式如下：{format_instructions}\n问题：{question}"
).partial(format_instructions=parser.get_format_instructions())

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.2)
chain = prompt | llm | parser

print(chain.invoke({"question": "如何解绑银行卡？"}))
```

**意图与边界**：确保可解析；边界是模型可能输出非法 JSON，需要重试。

## 常见坑与排错
- Prompt 模板里参数名不一致导致 KeyError。
- 模型输出 JSON 不规范，需要加重试或修复器。

## 性能/安全考虑
- 生产环境建议设置 **temperature ≤ 0.3**。
- 日志中避免存储用户敏感信息。

## 测试与验证
- 单元测试：PromptTemplate 是否格式化成功。
- 回归测试：同问题多次运行输出一致性。

## 最小可复现示例
1. 设置环境变量 `OPENAI_API_KEY`。
2. 运行示例 1。
3. 预期输出：三句话回答“如何重置密码”。


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
先搭建最小链路，确认可控输出，再逐步加入工具调用与检索，这是生产级 Agent 的安全起点。
