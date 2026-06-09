# Session 9 — Context / Handoff from Session 8

Paste this into (or point the new Session 9 chat at) the start of your Session 9
work. It tells the assistant everything it needs to know about the Session 8
foundation so it can be productive immediately. The actual Session 9 assignment
goes at the bottom (§10) — paste it there.

---

## 1. What I'm building (continuity)

This is the **EAG V3** course. Each session evolves the same AI agent:
- **Session 7** — a serial cognitive loop (Perception → Decision → Action → Memory) with FAISS-backed memory.
- **Session 8** — replaced the loop with a **growing multi-agent graph** (the agent loop IS a NetworkX DAG; ready nodes run in parallel via `asyncio.gather`).
- **Session 9 (now)** — adds **browser automation** via the `browser` skill that was stubbed and reserved in Session 8.

I want plain-English, step-by-step teaching as we work — define jargon in ordinary words; teach as you build, not just at the end.

## 2. The Session 8 architecture (the mental model)

- **The graph is the loop.** A **Planner** skill reads the query and emits a small graph of skill nodes. The **Executor** runs every node whose inputs are ready, in parallel. The graph **grows at runtime** from five sources: the Planner's seed plan, dynamic successors a skill emits, static `internal_successors` from YAML (e.g. `coder → sandbox_executor`), automatic **Critic** insertion after `critic:true` skills, and a recovery **Planner** added when a node fails.
- **A "skill" is just two files — no Python class.** A block in `agent_config.yaml` + a prompt `.md`. The engine loads skills generically. THIS IS KEY: you add a whole new capability (like Session 9's browser skill) by adding **data (config + prompt)**, not by editing the engine.
- **Persistence + resume.** Graph state is written to disk atomically (temp file + rename) after every step; a killed run resumes from the last boundary without repeating completed nodes.
- **Failure handling** lives in `recovery.py`: `transient` errors are skipped, `validation_error` is skipped, only `upstream_failure` triggers a re-plan.

## 3. Where everything lives

```
Session 8 .../                       ← repo root (git)
├── S8code/                          ← THE AGENT (our working copy). Run from here.
│   ├── flow.py        orchestrator (Graph + Executor + CLI) — DO NOT EDIT
│   ├── skills.py      skill registry + run_skill            — DO NOT EDIT
│   ├── recovery.py    failure classification + critic splice — DO NOT EDIT
│   ├── persistence.py / schemas.py / sandbox.py / mcp_runner.py / mcp_server.py — engine
│   ├── memory.py / vector_index.py / artifacts.py / perception.py / decision.py / action.py — S7 carryover, DO NOT EDIT
│   ├── agent_config.yaml   ← the skill catalogue (EDIT here to add a skill)
│   └── prompts/            ← one .md per skill (EDIT/ADD here)
│        coder.md  verifier.md (new in S8)  planner.md  critic.md  formatter.md
│        researcher.md  retriever.md  distiller.md  summariser.md  sandbox_executor.md
│        browser.md   ← STUB reserved for Session 9 (currently one line)
└── S8SharedCode/
    └── gateway/         ← LLM Gateway V8 (FastAPI), runs on :8108. Has the .env.
```

GitHub: **https://github.com/ashwanibindroo-personal/llm_gatewayv8-Multi-Agent-DAG-Orchestration**
Read `README.md`, `LEARNING_NOTES.md`, and `RUN.md` in the repo root for the full Session 8 writeup.

## 4. How to run (Windows PowerShell)

The agent code is NOT a sibling of the gateway, so you must tell it where the gateway is via an env var.

```powershell
# Terminal 1 — gateway (leave running)
cd "C:\The School Of AI\Session 8 - Multi-Agent DAG Orchestration (GRAPHS!)\S8SharedCode\gateway"
uv run main.py                       # boots on http://localhost:8108

# Terminal 2 — agent
cd "C:\The School Of AI\Session 8 - Multi-Agent DAG Orchestration (GRAPHS!)"
$env:EAGV3_GATEWAY_DIR = (Resolve-Path ".\S8SharedCode\gateway").Path
cd .\S8code
uv run python flow.py "your query here"
uv run python replay.py <session-id>   # walk a saved run
```

Prereqs: Python 3.11+, `uv`, Ollama with `nomic-embed-text` pulled, and `S8SharedCode\gateway\.env` with provider keys.

## 5. The skill catalogue (current)

planner · researcher (web) · retriever (knowledge base) · distiller (`critic:true`) ·
summariser · critic · **verifier (added in S8)** · coder · sandbox_executor · formatter ·
**browser (STUB — Session 9 fills this in)**.

**The Verifier is the worked example to copy for the Session 9 browser skill.** It was added with ONLY: `prompts/verifier.md` + a block in `agent_config.yaml` + a few lines of Planner/Formatter prompt guidance — and ZERO engine edits (all 17 engine `.py` files stayed byte-identical). Follow the same recipe for `browser`.

## 6. The gateway (V8 + Anthropic)

- Gateway V8 on port 8108: per-skill tagging, cost-by-agent endpoint, direct routing via `agent_routing.yaml`, built-in 5xx retries.
- **Anthropic/Claude was added** as a worker provider (adapter in `providers.py`, registered in `router.py` LIMITS+SHORTCUTS and `main.py` ORDER+TIER_TO_ORDER). Default model `claude-haiku-4-5-20251001` (set in `.env`).
- `agent_routing.yaml` routes cognitive skills → anthropic, critic/retriever → groq/anthropic. To route a NEW skill, add one line `skillname: provider`.

## 7. Conventions / rules to keep

1. **Never edit the engine** (`flow.py`, `skills.py`, `recovery.py`, schemas/persistence/sandbox, or the S7 carryover files). New skills = YAML + prompt only.
2. A skill's output is JSON; the prompt must specify the exact shape (no markdown fences).
3. If a skill needs a fixed follow-on (like browser → something), use `internal_successors` in the YAML.
4. To add MCP tools a skill can call, they're declared in `skills.py`'s `_TOOL_CATALOG` and exposed via `tools_allowed` in the YAML — **note:** adding a genuinely new tool may require touching `mcp_server.py`/`skills.py`, which is the one place Session 9 might legitimately extend the engine (confirm against the S9 assignment).

## 8. Gotchas learned in Session 8 (will save you time)

- **`memory.remember … 502 … falling back to fact-write`** — harmless. `memory.py` hard-pins its classifier to Gemini (`provider="g"`), and Gemini's free tier is ~20 requests/day; once exhausted it 429s (shown as 502). Memory still works via the deterministic fallback. Left untouched on purpose (it's carryover; also a teaching example of why config-driven routing beats hard-coded pins).
- **Windows file-lock on save** (`WinError 5: Access is denied` during `os.replace`) — Windows Defender/Search-indexer scanning the state files mid-rename. Fix once: run PowerShell **as admin** →
  `Add-MpPreference -ExclusionPath "C:\The School Of AI\Session 8 - Multi-Agent DAG Orchestration (GRAPHS!)"`.
- **`I/O operation on closed pipe`** tracebacks during web/tool runs — cosmetic Windows artifact of the MCP subprocess shutting down. Ignore.
- **Memory pollution changes Planner routing.** Re-running the same query makes the Planner reuse cached answers (via retriever) instead of fresh research. To force a clean run, empty memory:
  `Set-Content .\S8code\state\memory.json "[]" -Encoding utf8 ; Remove-Item .\S8code\state\index.faiss,.\S8code\state\index_ids.json -EA SilentlyContinue`
- **LLMs can't count.** The S8 critic accepted a 10-word answer to an "exactly 8 words" task. Mechanical checks (word counts, exact arithmetic) should be grounded in the Coder/sandbox, not judged by an LLM.
- **Run Python unbuffered** (`python -u` or `$env:PYTHONUNBUFFERED=1`) when capturing logs to a file, or buffered stdout gets lost on Ctrl-C.

## 9. Session 9 is browser automation

The `browser` skill (currently a one-line stub at `S8code/prompts/browser.md`, and a stub entry in `agent_config.yaml`) is where Session 9 lives. Expect it to let the agent drive a real browser (navigate, click, fill, extract) — likely via an MCP tool or a library like Playwright. Build it the same data-driven way the Verifier was built; only touch engine/tool plumbing if the assignment genuinely requires a new tool wired through `mcp_server.py`.

## 10. SESSION 9 ASSIGNMENT  ← paste the actual assignment text here

> (Paste the Session 9 assignment / course page here. Once it's in, the
>  assistant should: confirm scope, propose a design, then build the browser
>  skill + any required tool wiring, run + capture log-verified traces, and
>  produce the same deliverable set as Session 8: README, RUN.md,
>  LEARNING_NOTES.md, YouTube script, narration, traces/.)

---

### TL;DR for the new session
"I'm on Session 9 of EAG V3, continuing the Session 8 multi-agent DAG agent in
`S8code/`. Skills = YAML + prompt, never edit the engine. Session 9 fills in the
reserved `browser` skill (browser automation). Copy the Verifier as the template.
Gateway runs from `S8SharedCode/gateway` on :8108; set `$env:EAGV3_GATEWAY_DIR`
before running. Teach me step by step in plain English. The assignment is in §10."
