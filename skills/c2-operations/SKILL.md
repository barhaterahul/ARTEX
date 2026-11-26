---
name: c2-operations
description: 'Use to turn a one-shot command execution into a durable, in-memory operating session for post-exploitation.'
mcps: pa-c2
---
Authorized targets only. Doctrine. (1) Catch the callback / open the session through the pa-c2 MCP and upgrade to a full in-memory agent. (2) Run post-exploitation from memory rather than repeatedly staging tools on disk (lower EDR exposure). (3) One session per task; reserve an operator session before driving it. (4) Use payloads produced by the pipeline by reference; never inline a payload body. (5) Confirm the session with an identity command and clean up handlers/sessions when done. MITRE T1059, T1055.