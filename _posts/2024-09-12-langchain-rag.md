---
layout:       post
title:        "03. RAG 入门（初学）：让客服 Agent 会查资料"
author:       "iehmltym（張）"
header-style: text
catalog:      true
tags:
    - LangChain
    - RAG
    - 入门
---

面向读者：**初学**。场景：**生产级客服 Agent 查询知识库**。

## 背景/问题
直接让模型回答常常“编造”。RAG 通过检索文档让回答有据可依。

## 核心概念
- 文档切分
- 向量检索
- 检索结果注入 Prompt

## 方案设计
- 先把 FAQ 切分为小块
- 用向量库检索相关片段
- 用严格提示词要求“只基于资料回答”

## 关键实现（含代码）
**示例 1：构建最小向量库（可运行）**

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS

texts = ["重置密码：进入设置-安全-重置", "解绑银行卡：进入钱包-卡管理"]
chunks = RecursiveCharacterTextSplitter(chunk_size=50, chunk_overlap=5).split_text("\n".join(texts))

emb = OpenAIEmbeddings()
store = FAISS.from_texts(chunks, emb)
print(store.similarity_search("如何解绑银行卡？", k=1)[0].page_content)
```

**意图与边界**：快速检索；边界是没有权限控制。

**示例 2：把检索结果注入回答（可运行）**

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import PromptTemplate

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.2)
context = "解绑银行卡：进入钱包-卡管理"

prompt = PromptTemplate.from_template(
    "只基于资料回答。资料：{context}\n问题：{question}"
)

print((prompt | llm).invoke({"context": context, "question": "怎么解绑银行卡？"}).content)
```

**意图与边界**：约束模型；边界是资料不足时需回退策略。

## 常见坑与排错
- 切分过小导致语义破碎。
- 检索结果为空却仍生成答案。

## 性能/安全考虑
- FAQ 可做缓存，减少重复检索。
- 文档必须做权限隔离。

## 测试与验证
- 用已知问答对做检索准确率测试。
- 监控“空召回率”。

## 最小可复现示例
1. 设置 `OPENAI_API_KEY`。
2. 运行示例 1 和 2。
3. 预期输出：回答来自检索资料。


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
RAG 是生产级客服 Agent 的“事实来源”，但检索质量决定最终效果。
