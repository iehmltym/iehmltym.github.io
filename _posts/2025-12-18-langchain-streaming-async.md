---
layout:       post
title:        "【AI Agent】18. 流式与异步（高级）：降低延迟与提升吞吐"
author:       "iehmltym（張）"
header-style: text
catalog:      true
tags:
    - LangChain
    - 流式
    - 高级
---

面向读者：**高级**。场景：**生产级客服 Agent 并发服务**。

## 背景/问题
实时对话需要低延迟，但同步调用会阻塞。需要流式和异步调用。

## 核心概念
- Streaming
- Async invoke
- 并发控制

## 方案设计
- 对用户采用流式输出
- 内部任务用异步批处理

## 关键实现（含代码）
**示例 1：异步调用（可运行）**

```python
import asyncio
from langchain_openai import ChatOpenAI

async def main():
    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.2)
    resp = await llm.ainvoke("用一句话回答：如何退款？")
    print(resp.content)

asyncio.run(main())
```

**意图与边界**：异步调用模型；边界是需要事件循环。

**示例 2：并发批处理（可运行）**

```python
import asyncio
from langchain_openai import ChatOpenAI

async def ask(q):
    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.2)
    return (await llm.ainvoke(q)).content

async def main():
    results = await asyncio.gather(
        ask("退款多久到账？"),
        ask("怎么解绑银行卡？")
    )
    print(results)

asyncio.run(main())
```

**意图与边界**：并发处理；边界是并发过高会触发限流。

## 常见坑与排错
- 未限制并发导致速率限制。
- 异步异常未捕获导致任务中断。

## 性能/安全考虑
- 设置并发上限。
- 对流式输出进行内容过滤。

## 测试与验证
- 压测并发下的响应时间。
- 模拟限流错误场景。

## 最小可复现示例
1. 配置 `OPENAI_API_KEY`。
2. 运行示例 1。
3. 预期输出：一句话回答。


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
异步与流式是生产级 Agent 低延迟的关键手段。
