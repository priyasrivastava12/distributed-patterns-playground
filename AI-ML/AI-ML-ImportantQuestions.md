# AI/ML Important Questions (Interview Q&A)

This file contains practical, interview-ready answers for common GenAI software engineering questions.

## Core Terms (Quick Meaning)
- LLM: Large language model trained for next-token prediction.
- Token: Unit of text processed by model; drives cost and latency.
- Context window: Max tokens in one request.
- Embedding: Vector representation of semantic meaning.
- Vector DB: Similarity-search database for embeddings.
- RAG: Retrieve relevant context first, then generate answer.
- Fine-tuning: Additional training for domain-specific behavior.
- Guardrails: Safety and policy controls.
- Hallucination: Fluent but incorrect model output.
- MCP: Standard protocol to connect model to tools/context.

## Q1-Q8 Fundamentals
1. **What is Generative AI?**
   - Models that generate new content (text/code/images) from learned patterns.
2. **What are LLMs and how are they trained?**
   - Pretraining on large corpora + alignment/instruction tuning.
3. **Transformer architecture importance?**
   - Attention enables strong context understanding and scalable training.
4. **Attention mechanism?**
   - Lets model focus on relevant tokens for each prediction.
5. **Tokenization?**
   - Splits input into model-usable units; affects cost/window.
6. **Context window limitation?**
   - Large inputs must be chunked/summarized/retrieved.
7. **Temperature/top-k/top-p?**
   - Sampling controls for randomness and determinism.
8. **Embeddings and importance?**
   - Enable semantic search and retrieval systems.

## Q9-Q16 RAG
9. **Explain RAG.**
   - Retrieve trusted context from knowledge base before calling LLM.
10. **RAG components?**
   - Ingestion, chunking, embeddings, vector index, retriever, prompt builder, LLM.
11. **Vector search works how?**
   - Similarity match between query embedding and chunk embeddings.
12. **Chunking importance?**
   - Better retrieval precision and context fit.
13. **Retrieval techniques?**
   - Semantic, BM25, hybrid (most practical).
14. **How evaluate RAG?**
   - Relevance, faithfulness, latency, cost, user success.
15. **RAG limitations?**
   - Retrieval quality dependency + infra overhead.
16. **Improve retrieval?**
   - Better chunking, reranker, metadata filters, hybrid search.

## Q17-Q21 Prompt Engineering
17. Prompt techniques: zero-shot, few-shot, role-based, structured output.
18. Role-based prompting: improves output relevance and style.
19. Production prompts: role + task + constraints + output schema.
20. Hallucination reduction: require citations and "insufficient data" fallback.
21. Prompt injection: malicious instructions; mitigate via sanitization and policies.

## Q22-Q28 Practical Developer Usage
22. Daily usage: design, coding, debugging, tests, docs.
23. Debugging with LLM: provide logs + minimal repro + expected behavior.
24. Test generation: define edge cases and assert behaviors.
25. Token optimization: concise context + retrieval + output limits.
26. Large codebase handling: retrieval/indexed context, not full repo paste.
27. Cursor-like internals: IDE context + retrieval + model + patch loop.
28. IDE integration: plugin + policy + observability.

## Q29-Q32 Fine-tuning vs RAG
29. Fine-tune when behavior/format must be highly consistent.
30. Use RAG when knowledge changes often.
31. Combine both: fine-tune style, RAG facts.
32. Fine-tuning challenges: data quality, drift, cost, governance.

## Q33-Q38 Agents and MCP
33. AI agents: LLM + tools + planning.
34. Orchestration: multi-agent control with retries/state.
35. Chaining: planner -> coder -> tester -> reviewer.
36. Tool calling: structured API execution from model output.
37. MCP: standardized model-tool-context integration.
38. Autonomous design: permissions, fallback, human approvals.

## Q39-Q44 System Design
39. ChatGPT-like: API gateway, session memory, model router, moderation.
40. Copilot-like: IDE context + low-latency code model + retrieval.
41. AI code review: PR diff + static checks + LLM comments + policy gate.
42. Doc Q&A: ingestion + embedding index + retriever + cited answers.
43. Support bot: intent + RAG + live tool calls + escalation.
44. Scale LLM apps: caching, routing, queues, retrieval tuning.

## Q45-Q49 Performance
45. Reduce latency: streaming, smaller prompts/models, caching.
46. Reduce cost: token control, caching, model tiering.
47. Caching: avoid recomputation for repeated queries.
48. Batching: improve throughput and GPU usage.
49. Streaming: better perceived responsiveness.

## Q50-Q53 Evaluation
50. Evaluate output: faithfulness, relevance, correctness.
51. Core metrics: task success, hallucination rate, p95 latency, cost/request.
52. Hallucination detection: source checks + contradiction checks.
53. Testing: golden prompts, regression sets, adversarial tests.

## Q54-Q57 Security
54. Risks: leakage, hallucination, bias, unsafe tool actions.
55. Prompt injection: adversarial override attempts.
56. Secure apps: policy layer, RBAC, audit logs, output filters.
57. Sensitive data: PII redaction, encryption, controlled retention.

## Q58-Q64 Advanced
58. Multimodal AI: text + image + audio capabilities.
59. LoRA/PEFT: efficient adapter-based fine-tuning.
60. Quantization: lower precision for cost/latency gains.
61. Distillation: train smaller model from larger teacher.
62. Agent memory: short-term + long-term state.
63. Long-term vs short-term memory: persistent vs session context.
64. Self-reflection: model critiques and improves own answer.

## Q65-Q70 Scenario Answers
65. Wrong answers -> improve retrieval, citations, and fallback handling.
66. High cost -> cache, route smaller models, trim prompts.
67. Slow response -> stream, optimize retrieval, reduce context.
68. Real-time data -> combine RAG with live API tool calls.
69. Private enterprise AI -> VPC/local deployment + strict ACL.
70. Prevent leaks -> DLP controls, redaction, policy enforcement.

## Strong Closing Line
> I combine prompt engineering, RAG, and controlled tool use; then optimize with retrieval quality, caching, and model routing while enforcing security, compliance, and human approval for high-risk decisions.
