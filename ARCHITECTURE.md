# Architecture Note — Session 9 Browser Agent

This is the brief architecture note required by the assignment. It explains how
the browser capability plugs into the Session 8 orchestrator without modifying
it, how the cascade works, and what was built versus ported.

## 1. The orchestrator is frozen; capability is data

The Session 8 engine is a **growing-graph orchestrator**. The Planner emits a
small graph of skill nodes; the Executor runs every node whose inputs are ready
(in parallel via `asyncio.gather`); the graph grows at runtime from five
sources — the Planner's seed plan, dynamic successors a skill returns, static
`internal_successors` from YAML, automatic Critic insertion after `critic:true`
skills, and recovery re-planning when a node fails. There is no hand-written
`while` loop: the "agent loop" is an emergent property of the graph shape.

A **skill is two files, not a Python class**: a block in `agent_config.yaml` +
a prompt `.md`. The engine loads skills generically. Adding a capability means
adding **data**, not editing the engine.

**Invariant honored:** `flow.py` is byte-identical to the Session 8 engine.
The browser capability is added entirely through (a) data — YAML + prompts —
and (b) two pre-existing extension seams described below. No agent framework
(LangChain, CrewAI, etc.) is used; the browser logic is hand-written over
Playwright.

## 2. How the Browser skill plugs in — the `skills.py` seam

`skills.py` dispatches most skills by rendering their prompt and calling the
gateway. Two skills bypass that LLM channel and run dedicated code instead:
`sandbox_executor` (runs code) and — Session 9 — `browser`. The browser branch
builds a `NodeSpec` from the node's `metadata` (`url`, `goal`) and hands off to
`BrowserSkill.run()`, which returns a typed `BrowserOutput`. This is the same
shape the engine already used for `sandbox_executor`, so the orchestrator never
learns what a browser is — it just schedules a node and receives an
`AgentResult`.

The skill's output schema (`BrowserOutput` in `schemas.py`) carries `path` —
the cascade layer that actually ran — plus `actions`, `content`, `final_url`,
and `turns`. A structured `ErrorCode` (`gateway_blocked`, `interaction_failed`,
…) lets recovery branch on browser-specific failures while leaving every other
skill's text-heuristic failure path untouched.

## 3. The four-layer cascade (`browser/skill.py`)

A cost ladder — try the cheapest sense first, escalate only when the layer's
output is empty or the goal demands interaction:

1. **extract** — `httpx` GET + `trafilatura` text extraction. No browser, no
   LLM. Rejected when content < ~200 chars, or when the goal contains
   interactive verbs (`filter`, `sort`, `click`, …) that extraction can never
   satisfy.
2. **deterministic** — replays caller-supplied `metadata.selectors` via
   Playwright. Only fires when selectors are given; we never guess them.
3. **a11y** — opens real Chromium, reads the page's **accessibility tree** into
   a numbered legend (`dom.py`), and a text-only LLM picks one action per turn
   (`A11yDriver` in `driver.py`). Cheapest interactive layer — no image tokens.
4. **vision** — screenshots annotated with numbered boxes (`highlight.py`) sent
   to the gateway's `/v1/vision`; a vision LLM answers with a box number
   (`SetOfMarksDriver`). The expensive last resort.

**blocked** is orthogonal: at every layer the skill sniffs the page for
CAPTCHA / Cloudflare / login-wall / gated-model markers and, on a match,
returns `error_code="gateway_blocked"` immediately. The recovery path reads
that and re-plans around the dead URL rather than burning the step budget.

Drivers run a bounded loop (step cap, consecutive-failure cap, wall clock) and
write per-turn evidence — `turn_NN_raw.png`, `turn_NN_marked.png`,
`turn_NN_legend.txt` — under `state/sessions/<sid>/browser/`. That evidence is
both the assignment's deliverable #5 and the debugging instrument we used to
diagnose every demo failure.

## 4. The gateway (`llm_gatewayV9`, port 8109)

The agent never talks to an AI vendor directly; it calls the gateway, which
owns provider selection, rate limits, failover, and a per-agent cost ledger.
Session 9 work here:

- **Ported the Anthropic provider** from the Session 8 gateway into V9
  (`providers.py`): Claude's Messages API differs from OpenAI's (auth header,
  tool shape, system as a top-level field, structured output emulated via a
  forced tool call). The one genuinely new code path is **multimodal
  translation** — converting V9's canonical image blocks into Anthropic's
  `{"type":"image","source":{"type":"base64",…}}` shape so the vision layer's
  screenshots reach Claude. A guard falls through to the plain path when image
  blocks yield no data, avoiding Anthropic's empty-content 400.
- **Registered it** across `router.py` (limits + shortcuts), `main.py`
  (failover order + tier map), and `pricing.py` (Claude Haiku 4.5 at
  $1/$5 per Mtok, for the cost report).
- **Routing pins** (`agent_routing.yaml`): cognitive skills → Anthropic; the
  browser driver layers → Anthropic (vision-capable, 100 rpm — a hard-pinned
  Gemini buckled under parallel browser bursts on its 15 rpm free tier).

## 5. The replay viewer (`replay_viewer.py`)

A standalone report generator (zero engine changes). It reads the persisted
session (`query.txt`, `graph.json`, per-node JSON via `SessionStore`), the
browser artifact folders, and the gateway's `/v1/cost/by_agent` ledger, and
emits one self-contained `replay.html` — all CSS inline, screenshots
base64-embedded, DAG drawn with dependency-free layered HTML — covering all
eight required trace items. It fails loudly on an unknown session id rather
than writing an empty report.

## 6. What was built vs. ported vs. untouched

| | |
|---|---|
| **Built (ours)** | `replay_viewer.py`; browser demo-mode env vars (`BROWSER_HEADFUL`/`BROWSER_SLOWMO_MS`) + new-headless channel + `goto` retries; the Anthropic→V9 multimodal port; planner/distiller/driver prompt engineering for the two-then-one-wave plan; routing + pricing config |
| **Ported (from S8)** | the `verifier` skill (prompt + YAML) and the real `coder` prompt; the Anthropic provider class |
| **Instructor base (reused)** | the browser cascade scaffolding (`driver.py`, `dom.py`, `highlight.py`, `client.py`, and the `skill.py` skeleton), the V9 gateway, the S8 engine |
| **Untouched (frozen)** | `flow.py`, `recovery.py`, `persistence.py`, `schemas.py` core, and the S7 carryover — the assignment's no-orchestrator-edits invariant |

## 7. Worked example — the golden trace (`traces/s8-5d301d39/`)

Query: *"Compare top 3 Hugging Face text-generation models sorted by likes."*
The seed plan was three nodes (browse list → distill all fields → format).
The first browses did not reliably apply the **likes** sort, so the
auto-inserted **Critic** rejected the distiller's output — correctly noting the
order reflected a downloads sort — and recovery re-browsed. After three such
cycles the browse genuinely applied the likes sort (13.4k > 6.57k > 6.07k), the
Critic passed, and the single formatter produced the final table. One formatter
in the whole graph means the engine's "last formatter wins" rule yields a
deterministic, correct answer. The cascade ran entirely on the **a11y** layer;
license is "see model card" because it is not shown on listing cards (detail
pages were deliberately avoided — gated models like Llama wall them off).
