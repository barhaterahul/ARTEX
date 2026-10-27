---
name: ai-prompt-injection
description: 'Use when a target exposes an LLM chat, assistant, or tool-using agent and you can influence any text that reaches the model (user input, uploaded documents, fetched web content, tool output, or a shared store it reads). Tests prompt injection, system-prompt disclosure, guardrail bypass, goal hijacking, and agent-memory or cross-session exposure, exercised via the pa-ai-target MCP.'
mcps: pa-ai-target
---
Authorized targets only. Use the pa-ai-target MCP: ai_chat to probe the assistant and map its tools and refusal behavior; ai_inject(endpoint, marker, corpus) to run an injection corpus and score each response against a unique marker; ai_score and ai_agent_log as the confirmation oracle. Treat a finding as real only when the marker is observed, never on model judgement. Keep payloads in the ai-target-server corpus file and reference it by the corpus path; do not inline payloads.