---
layout:       post
title:        "【AI Agent】02. LCEL 组合式编排（初学）：让流程可读、可替换"
author:       "iehmltym（張）"
header-style: text
catalog:      true
tags:
    - LangChain
    - LCEL
    - 入门
---

面向读者：**初学**。场景：**生产级客服 Agent 的标准化流程**。

## 背景/问题
当流程变复杂（分类、检索、总结），把逻辑写进一个函数会失控。需要可视化的管道式表达。

## 核心概念
- **Runnable**：可执行节点。
- **管道操作符 `|`**：组合流程。
- **并行 `map`**：批量处理。

## 方案设计
把流程拆成模块：输入规范化 → LLM → 解析 → 审计日志。

## 关键实现（含代码）
**示例 1：最小 LCEL 链（可运行）**

```python
from langchain_core.prompts import PromptTemplate
from langchain_openai import ChatOpenAI

prompt = PromptTemplate.from_template("用一句话总结：{text}")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.2)
chain = prompt | llm

print(chain.invoke({"text": "用户忘记密码，想自助重置。"}).content)
```

**意图与边界**：快速验证管道可运行；边界是无后处理。

**示例 2：加入解析器（可运行）**

```python
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import PromptTemplate
from langchain_openai import ChatOpenAI

prompt = PromptTemplate.from_template("请输出一句话结论：{text}")
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.2)
parser = StrOutputParser()
chain = prompt | llm | parser

print(chain.invoke({"text": "用户申诉被误封"}))
```

**意图与边界**：保证输出为纯文本；边界是无法结构化。

## 常见坑与排错
- LCEL 里输入输出类型不一致导致运行错误。
- 多环节链路难定位，可对每步单独 `invoke`。

## 性能/安全考虑
- 在生产里可开启缓存，减少重复调用。
- 提示词包含用户隐私时要脱敏。

## 测试与验证
- 单测每个节点是否返回正确类型。
- 在 CI 里跑最小链路样例。

## 最小可复现示例
1. 配置 `OPENAI_API_KEY`。
2. 运行示例 1。
3. 预期输出：一句话总结。


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
LCEL 让链条像“流水线”，可读、可替换，是生产 Agent 的基础表达方式。
