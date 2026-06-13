# EAG V3 · Session 9 — Browser Agent on a Growing-Graph Orchestrator

A multi-agent agent that **drives a real web browser** to answer a comparison
question, built on the Session 8 growing-DAG orchestrator. The agent files a
plan as a graph of skills, runs ready nodes in parallel, grows the graph at
runtime (dynamic successors, auto-inserted critics, recovery re-planning), and
— new in Session 9 — reaches the live web through a **four-layer browser
cascade**.

> **Assignment task:** *"Compare the top 3 Hugging Face text-generation models
> sorted by likes."* The agent filters the models listing by task, sorts by
> likes, and extracts a structured comparison table — performing several
> **visible** browser interactions (filter, open sort dropdown, sort) along the
> way.

**▶ Demo video:** https://youtu.be/HG5miTC_bvA

## The browser cascade (the core idea)

The Browser skill walks a **cost ladder** — cheapest sense first, escalating
only when forced:

```
Layer 1  extract        plain HTTP GET + trafilatura text   (no browser, no LLM)
Layer 2a deterministic  replay caller-supplied CSS selectors (Playwright)
Layer 2b a11y           read the page's accessibility tree, a text LLM picks actions
Layer 3  vision         numbered-box screenshots, a vision LLM picks actions
   ·     blocked         CAPTCHA / login / gated-model wall → first-class failure
```

A goal containing interactive verbs ("filter", "sort", "click") forces
escalation past Layer 1. `blocked` is not a crash — it returns
`error_code="gateway_blocked"`, which the orchestrator's recovery path turns
into a re-plan. Every layer the agent actually used is reported in the trace.

## Architecture at a glance

Three cooperating programs:

```
 your query
     │
     ▼
┌──────────────┐   "I need an LLM to think"   ┌────────────────────┐
│   S9code/    │ ───────────────────────────▶ │   llm_gatewayV9/    │
│  THE AGENT   │ ◀─────────────────────────── │   THE GATEWAY       │
│ (brain+hands)│     answers + token bills     │ (switchboard to     │
└──────┬───────┘                               │  Anthropic/Gemini…) │
       │ "I need to USE a web page"            │  :8109, /v1/vision  │
       ▼                                       └────────────────────┘
┌──────────────────┐
│ S9code/browser/  │  drives a real Chromium window
└──────────────────┘
```

The orchestrator (`flow.py`) is **never edited** — a hard assignment
invariant. New capability is added as **data** (a YAML block + a prompt) or
behind existing seams (the `skills.py` browser dispatch, the gateway's provider
registry). See `ARCHITECTURE.md` for the full account of what was built vs.
ported.

## The 8 deliverables — where each one lives

The assignment requires a replay log tracking eight things. `replay_viewer.py`
renders all eight into one self-contained `replay.html` per run:

| # | Item | Where it comes from |
|---|------|---------------------|
| 1 | Original user goal | `state/sessions/<sid>/query.txt` |
| 2 | Planner DAG | `graph.json` → layered HTML diagram with cascade badges |
| 3 | Browser path (extract/deterministic/a11y/vision/blocked) | `BrowserOutput.path` / `error_code` per node |
| 4 | Exact actions taken | `BrowserOutput.actions` (per-turn action log) |
| 5 | Evidence (screenshots + page-state) | `browser/**/turn_NN_*.png` + `*_legend.txt`, embedded |
| 6 | Raw extracted data | `BrowserOutput.content` per node |
| 7 | Final comparison table | the single formatter node's `final_answer` |
| 8 | Turn count + cost summary | per-node turn counts + gateway `/v1/cost/by_agent` ledger |

A committed golden trace lives in **[`traces/s8-5d301d39/`](traces/)** — open
`replay.html` in any browser (no network needed).

## Repository layout

```
.
├── S9code/                 ← THE AGENT (working copy of the instructor base)
│   ├── flow.py             orchestrator — FROZEN, never edited
│   ├── skills.py           skill loader + browser dispatch seam
│   ├── agent_config.yaml   skill catalogue (+ our verifier block)
│   ├── prompts/            one .md per skill (planner/distiller/… edited here)
│   ├── browser/            the cascade: skill.py, driver.py, dom.py, highlight.py, client.py
│   ├── replay_viewer.py    ← NEW: 8-item self-contained replay.html generator
│   └── tests/              pytest suite (34 passing offline)
├── llm_gatewayV9/          ← THE GATEWAY (:8109) + our ported Anthropic provider
├── traces/                 ← committed golden submission run (replay.html + JSON)
├── docs/superpowers/       design spec + implementation plan
├── RUN.md                  how to run it (two-terminal PowerShell)
├── ARCHITECTURE.md         the brief architecture note
├── LEARNING_NOTES.md       plain-English teaching recap + debugging arc
└── YOUTUBE_SCRIPT.md       scene-by-scene demo script
```

## Quick start

See **[RUN.md](RUN.md)** for the full two-terminal flow. The short version:

```powershell
# Terminal 1 — gateway
cd llm_gatewayV9 ; uv run main.py            # boots on :8109

# Terminal 2 — agent (visible browser)
cd S9code
$env:BROWSER_HEADFUL = "1"; $env:BROWSER_SLOWMO_MS = "400"
uv run python flow.py "Compare top 3 Hugging Face text-generation models sorted by likes"
uv run python replay_viewer.py <session-id>   # → state/sessions/<id>/replay.html
```

## Constraints honored

- **No orchestrator edits** — `flow.py` is byte-identical to the Session 8 engine.
- **No agent frameworks** — Playwright + our own logic; no LangChain/CrewAI/etc.
- **≥3 visible interactions** — task filter, sort dropdown, sort selection on the
  live Hugging Face models page.
