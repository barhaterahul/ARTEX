# ARTEX ← PEN-AGENT post-exploitation arsenal

Status: **applied (MCPs disabled, skills isolated to a test agent).** Adds the
internal/post-exploitation half ARTEX lacks — Windows/AD privilege escalation,
Active Directory attacks, C2 sessions, and pivoting — by wiring in PEN-AGENT's
post-ex MCP servers and porting its tradecraft as skills. Same mechanism as the
AI-attack suite (`docs/ai-attack-integration.md`): a thin `SKILL.md` gates an
MCP server via `mcps:`.

## 1. The gap this closes

ARTEX's entire skill library (api-recon, api-security, bbot-recon, codeaudit,
patch-diff, php2shell, playwright, rea, scopesentry) is **edge-to-initial-access**:
web, API, recon, RE, source audit. It has **nothing for the post-exploitation
half of the kill chain** — the half where a hardened internal target is won or
lost. On a head-to-head lab run, this is exactly where ARTEX stalled and
PEN-AGENT succeeded:

| PEN-AGENT won | ARTEX's problem | Skill added | MCP gated |
|---|---|---|---|
| labDB `SeImpersonate→SYSTEM` (in-process SQLCLR potato) | dropped stock potato EXEs → **Defender quarantined all 4** | `windows-privilege-escalation` | `pa-c2` |
| DC access + spotted ADCS ESC / RBCD primitives | no AD tradecraft at all | `active-directory-attacks` | `pa-ad`, `pa-bloodhound` |
| Durable Meterpreter sessions, in-memory post-ex | one-shot tool runs only | `c2-operations` | `pa-c2` |
| ligolo SOCKS pivot into the internal segment | crude pivoting, no doctrine | `lateral-movement-pivoting` | `pa-ligolo` |

The sharpest example: on labDB, ARTEX reached the same `svc_sql` /
SeImpersonate foothold but lost SYSTEM 0/4 because it **staged tool binaries to
disk** (GodPotato, PrintSpoofer, packed/renamed recompiles — all AV-quarantined
on Server 2025). PEN-AGENT won by running the escalation **in-process via
SQLCLR** inside `sqlservr.exe` and choosing the DCOM/OXID coercion vector that
still works on that build. That doctrine is now the core of
`windows-privilege-escalation`. The lesson: against EDR, **technique quality
beats iteration speed** — tooling without doctrine is why ARTEX had `john` and
still lost.

## 2. What was added

Four MCP servers (created **disabled**), tools cached, connectivity proven:

| MCP | transport | points at | tools |
|---|---|---|---|
| `pa-ad` | stdio (`uv run … tools/ad-server`) | fresh process | kerberoast, asreproast, secretsdump, adcs_find, request_cert, auth_cert, smb_getfile |
| `pa-bloodhound` | stdio (`uv run … tools/bloodhound-server`) | BloodHound CE | bhce_cypher, bhce_shortest_path, bhce_asset_paths, bhce_mark_owned, bhce_status |
| `pa-c2` | **sse** → `http://127.0.0.1:8024/sse` | PEN-AGENT's live Metasploit daemon | run_module, execute, generate_payload, session/handler mgmt, spawn_operator_session, … (21) |
| `pa-ligolo` | **sse** → `http://127.0.0.1:8025/sse` | PEN-AGENT's live ligolo proxy | add_route, remove_route, start_proxy, stop_proxy, proxy_status |

The two SSE servers join PEN-AGENT's existing daemons **as a client** — no
duplicate process to contend with `msfrpcd` / the ligolo proxy.

Four gating skills (created **invisible to every agent**), payloads referenced
by staged path, never inlined — doctrine only. An isolated test agent
`postex_redteam_test` holds the four skills; the live planner/worker see only
`api-recon`, so nothing changes for a running task.

## 3. How ARTEX actually uses it — goal → intent → skill → fact

ARTEX runs a task as an **event-driven graph loop** over three primitives:

- **Goal** — a *final, verifiable deliverable*. The goal-decomposer
  (`agent/goals.go`) breaks the task description into discrete goal nodes (plus
  operation **constraints** and the authorization **scope**). Example goal:
  *"Escalate from database access to SYSTEM on the lab DB server."*
- **Intent** — a *unit of exploration/exploitation work*. The **Planner**
  (`agent/planner.go`) is the **sole intent generator**: whenever the asset or
  exploration graph changes, it reads the route, queries assets, judges whether
  each goal is met, and emits 0..N **intents** into the frontier. Each intent is
  dispatched to a **Worker**, and carries an `intent_id`.
- **Fact** — an *atomic piece of discovered knowledge*, anchored to an asset/
  node. Workers call `record_fact` as they learn things (a privilege held, a
  recovered secret, a confirmed behavior); a confirmed vuln becomes a **finding**
  (`add_finding`). Facts/findings mutate the graph, which wakes the Planner for
  the next intent.

**The skill is the connector between an intent and the MCP tools.** When a
Worker executes an intent, it picks a skill whose **`description`** matches the
situation; loading that skill (the model-invoked skill meta-tool) **unlocks the
MCP tools named in its `mcps:`** (deferred-tool gating in `agent/assembly.go` /
`agent/deferred.go`). So the flow for our arsenal is:

```
Goal: "SYSTEM on labdb01"
  └─ Planner emits Intent: "escalate from the sa foothold on labdb01"
       └─ Worker matches intent → loads skill `windows-privilege-escalation`
            └─ skill `mcps: pa-c2` → pa-c2 tools unlock
                 └─ Worker runs pa-c2 (generate_payload / run_module / execute),
                    guided by the skill's doctrine (in-process SQLCLR, DCOM/OXID
                    vector, no on-disk stock binaries)
                      └─ records Fact "SYSTEM on labdb01 (whoami oracle)" + Finding
                           └─ graph changes → Planner judges the goal met,
                              or emits the next intent (dump secrets → pivot → DC)
```

The skill's `description` is what makes the Worker *reach* for it on the right
intent; the skill's body is the doctrine that makes it pick the approach that
works; the `mcps:` field is what hands it the actual tools; and the **facts** it
records are how the Planner measures progress toward the **goal**.

## 4. Turning it on (two switches — both intentionally left off)

Nothing above runs in a live task yet, by design. Two switches enable it:

1. **Enable the MCP(s)** you need (`enabled=true`) — e.g. `pa-c2`. Gating in
   `assembly.go` requires the gated server be enabled before a loaded skill can
   connect it.
2. **Grant the skills to the executing agent.** Intents are executed by the
   **worker** agent, so the skills must be visible to it (and, for planning
   awareness, optionally the planner). They are currently visible only to the
   isolated `postex_redteam_test` agent.

Via the API (login → token first):

```
# enable an MCP (id from GET /api/mcp)
POST /api/mcp/{id}        {... "enabled": true}         # or the UI toggle
# make a skill visible to the live worker
POST /api/visibility/skill/toggle  {"agent_id":"<worker-id>","skill_name":"c2-operations","visible":true}
```

For a safe first run, assign a task to `postex_redteam_test` (which already has
all four skills) and enable only the MCP that task needs.

## 5. Safety, prerequisites, rollback

- **Scope is code-enforced.** `pa-c2` / `pa-ad` / `pa-ligolo` validate
  RHOSTS/subnets against PEN-AGENT's `scope.allow` and refuse out-of-scope
  targets — a *hard* guardrail ARTEX itself no longer has (its RoE guard was
  removed; scope is otherwise soft).
- **Schema-cached ≠ backend-ready.** A live run needs the backend up: `pa-c2` →
  `msfrpcd`; `pa-ligolo` → the ligolo proxy; `pa-ad` → a reachable DC + creds;
  `pa-bloodhound` → a running BloodHound CE **with data ingested**
  (`bhce_status` reports this).
- **Shared daemons.** `pa-c2` / `pa-ligolo` point at PEN-AGENT's own live C2 and
  tunnel; if a PEN-AGENT engagement is running, ARTEX shares that state.
- **Rollback** = disable/delete the MCP rows + remove the skill-visibility grants
  (or delete the skills). The live `browser` MCP and existing tasks are
  untouched throughout.

## 6. What persists where

Git-tracked (survives a rebuild): the four `skills/*/SKILL.md` + this doc.
Runtime DB state (not in git, recreate via the API): the four MCP rows, the
`postex_redteam_test` agent, and the skill-visibility grants.
