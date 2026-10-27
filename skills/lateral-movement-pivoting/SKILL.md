---
name: lateral-movement-pivoting
description: Use when a foothold host reaches a network segment you cannot reach directly and you need scoped routing into it.
mcps: pa-ligolo
---
Authorized targets only. Doctrine. (1) Confirm the far target is actually reachable FROM the foothold before building any tunnel (reach-gate) so you do not tunnel toward a dead host. (2) Establish the pivot through the pa-ligolo MCP (agent + scoped route) in preference to heavyweight in-framework SOCKS; prefer native SSH tunnels when you already hold SSH. (3) Keep routes scoped to in-scope CIDRs only and tear them down when the leg is done. MITRE T1090, T1572.