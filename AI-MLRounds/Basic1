# AI/ML Interview Guide for Software Engineers

## Purpose
This document is a production-ready reference for AI/ML interview rounds focused on software engineering roles. It covers practical application of LLMs in day-to-day development, architecture decisions, trade-offs, and interview-quality answer patterns.

## 1) Using LLMs Effectively During Development

### Why context matters
LLMs are context-driven and do not retain durable intent across unrelated requests. Better prompts produce more accurate and cheaper responses.

### Prompt template (recommended)
- Role: who the model should act as
- Task: what you want done
- Context: repo, language, constraints, expected behavior
- Output format: review checklist, patch, test plan, summary

### Example
"Act as a senior backend engineer. Review this service for scalability and reliability. Focus on database calls, concurrency, memory pressure, and failure handling. Return findings as: critical issues, medium issues, suggested fixes, and test cases."

## 2) Token Optimization and Cost Control

### Key principle
Cost and latency increase with prompt and response length. Keep context precise.

### Practical techniques
- Avoid repeating static context in every request
- Send only relevant code snippets, not full files by default
- Summarize logs before passing to the model
- Use retrieval (RAG) instead of pasting large documentation
- Bound output with explicit limits (for example: "max 8 bullets")

### Anti-pattern vs good pattern
- Anti-pattern: paste full repository context for a small bug
- Good pattern: provide one failing function, stack trace, and expected behavior

## 3) Agents, MCP, and Agent Chaining

### Agent
An agent combines model reasoning with tool execution (files, tests, APIs, queries).

### MCP (Model Context Protocol)
MCP standardizes model-to-tool integration so the model can access controlled external context such as repositories, databases, and issue trackers.

### Agent chaining pattern
- Planner: decomposes task
- Implementer: edits code
- Tester: validates behavior
- Reviewer: checks risk and regressions

### Example engineering workflow
1. Read PR
2. Run tests and static checks
3. Suggest or apply fixes
4. Produce review notes with risk level

## 4) Fine-Tuning vs Prompting vs RAG

### Decision guideline
Use the lowest-cost approach that meets quality requirements:
1. Prompting
2. Prompting + RAG
3. Fine-tuning (only when repeatedly needed)

### Use fine-tuning when
- You need consistent specialized output format
- Domain language is highly specific
- Prompting and RAG do not meet quality targets

### Avoid fine-tuning when
- Problem is mostly retrieval over changing documents
- Task is general coding assistance
- Data volume/quality is insufficient for training

## 5) Local LLM Setup for Repository Work

### Typical stack
- Runtime: Ollama or llama.cpp
- Optional desktop orchestration: LM Studio
- IDE/CLI integration for developer workflows

### Benefits
- Better data privacy for sensitive code
- Predictable cost
- Offline or low-network development support

### Trade-offs
- Smaller local models may reduce quality
- Infrastructure and maintenance overhead
- Throughput limits on weaker hardware

## 6) Prompt Engineering Standards

### Prompt structure
- Objective
- Constraints
- Input scope
- Acceptance criteria
- Output schema

### Useful styles
- Zero-shot: direct task without examples
- Few-shot: include examples for style/format control
- Structured response: enforce JSON/table/checklist outputs for automation

### Engineering tip
Request deterministic formats when output feeds downstream automation.

## 7) How LLMs Work Internally (Interview Level)

### Core concepts
- Transformer architecture
- Self-attention for token relationships
- Next-token prediction objective
- Context window limits

### Concise interview explanation
"An LLM tokenizes input, processes tokens through transformer layers with attention, and predicts the most likely next token repeatedly. It does not reason like humans; it predicts statistically plausible continuations based on training patterns and provided context."

## 8) Repository Guardrails with `claude.md` (or equivalent policy files)

### Purpose
A policy/config file defines coding and review expectations for AI-assisted changes.

### Example policy rules
- Follow project architecture boundaries
- Do not hardcode secrets or environment-specific values
- Add or update tests for behavioral changes
- Keep error handling explicit and observable
- Preserve backward compatibility unless approved

### Value
- Consistent code quality
- Repeatable AI behavior in reviews and generation
- Lower onboarding friction for contributors

## 9) Interview-Ready Answer Template

### Question: "How do you use AI in software development?"
"I use AI across the development lifecycle. During implementation and review, I provide structured prompts with role, task, and constraints so output is precise and actionable. I control cost and latency through token discipline and retrieval-based context instead of sending large raw inputs. For automation, I use agent workflows connected to tools for PR checks, testing, and documentation updates. For domain-heavy tasks, I evaluate prompting and RAG first, and only consider fine-tuning when quality requirements cannot be met consistently. For sensitive repositories, I use local models or strict data boundaries with policy-driven guardrails."

## 10) What Interviewers Usually Evaluate
- Practicality: can you apply AI in real engineering workflows
- Judgment: can you choose between prompting, RAG, and fine-tuning
- Reliability: can you validate outputs and reduce regressions
- Cost awareness: can you optimize tokens, latency, and model usage
- Security mindset: can you prevent data leakage and unsafe automation

## 11) Risk Management and Governance

### Common risks
- Hallucinated implementation details
- Over-trust in generated code
- Sensitive data exposure in prompts
- License/compliance issues for generated artifacts

### Mitigations
- Validate with tests and static analysis
- Keep human review mandatory for critical paths
- Mask or exclude sensitive data
- Enforce repository policies and audit trails

## 12) Evaluation Checklist for AI-Assisted Engineering
Use this checklist before accepting AI-generated output:
- Correctness verified by tests
- Edge cases covered
- Performance and scalability considered
- Security and privacy constraints met
- Logging/observability added where needed
- Documentation updated

## 13) Rapid Q&A (Short, Production-Ready)

### Q: When should you use RAG?
Use RAG when answers depend on changing internal knowledge (codebase, runbooks, product docs).

### Q: When is fine-tuning justified?
When repeated high-volume tasks require stable domain-specific behavior that prompting+RAG cannot reliably deliver.

### Q: How do you reduce token cost quickly?
Limit scope, remove redundant context, summarize inputs, and constrain output format and length.

### Q: How do you safely use AI in CI/CD?
Use policy files, restricted tool permissions, test gates, and human approval for high-impact changes.

## 14) Recommended Study Order
1. Prompt quality and token discipline
2. RAG fundamentals and retrieval quality
3. Agent workflows with tool integration
4. Local model deployment and privacy controls
5. Fine-tuning decision framework
6. Evaluation and governance practices

## 15) Final Notes
This guide is intentionally implementation-focused. In interviews, prioritize concrete examples, trade-offs, and validation practices over generic definitions.
