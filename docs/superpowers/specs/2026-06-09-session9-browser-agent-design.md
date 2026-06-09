# Session 9 — Browser Agent on the DAG Orchestrator (Design)

Date: 2026-06-09
Status: approved by user (brainstorming session)

## 1. Goal

Add browser automation to the EAG V3 multi-agent DAG agent (Session 8 lineage)
by filling in the reserved `browser` skill, then demonstrate it on a dynamic-web
comparison task with at least three visible browser interactions, and produce
the assignment's full deliverable set (structured comparison table, 8-item
replay viewer, video, repo, architecture note).

**Demo task (chosen):** "Compare the top 3 text-generation models on Hugging
Face sorted by downloads — report name, downloads, likes, license, and
parameter count." Hugging Face was chosen over e-commerce because it is
bot-friendly, stable, and naturally requires the mandated interactions
(apply task filter, sort, click into model pages).

## 2. Constraints (assignment invariants)

- `flow.py` (the orchestrator) is never modified.
- No agent frameworks (LangChain, CrewAI, etc.). Playwright + own logic only.
- New behaviour plugs in via the skill catalogue (`agent_config.yaml` +
  prompt) or as a direct extension of the Browser skill. The sanctioned
  engine-adjacent touch-points are `skills.py` dispatch, `schemas.py`
  (both already done in the instructor base) and the browser package itself.
- The agent must perform ≥3 visible interactions on a dynamic page
  (type / filter / sort / click into detail pages).

## 3. Decisions made

| Decision | Choice |
|---|---|
| Demo task | Hugging Face top-3 text-generation models by downloads |
| Codebase | Start from instructor base `S9SharedCode/code` → copy to `S9code/` |
| Gateway | `llm_gatewayV9` (port 8109, has `/v1/vision`) + port user's Anthropic adapter from V8 |
| Replay viewer | New standalone `replay_viewer.py` emitting one self-contained `replay.html` per run |
| Video visibility | `BROWSER_HEADFUL=1` + `BROWSER_SLOWMO_MS` env vars in the browser skill; default stays headless |

## 4. Repo layout

```
Session 9 - Browser Agents & Autonomous Web\   ← git repo root (this folder)
├── S9code\            ← working copy of S9SharedCode\code (instructor base)
├── llm_gatewayV9\     ← course gateway; receives the Anthropic port; runs on :8109
├── traces\            ← committed replay.html + raw JSON for the submission run
├── docs\superpowers\specs\  ← this spec + the implementation plan
└── README.md, RUN.md, LEARNING_NOTES.md, ARCHITECTURE.md, YOUTUBE_SCRIPT.md
```

`S9SharedCode\` remains pristine as the reference. `__MACOSX\` is junk —
gitignore it.

## 5. What the instructor base already provides (do not rebuild)

- `browser/` package: `client.py` (V9Client), `dom.py`, `driver.py`
  (A11yDriver, SetOfMarksDriver), `highlight.py`, `skill.py` (the cascade).
- The cascade in `browser/skill.py`:
  - Layer 1 **extract**: plain HTTP GET + trafilatura text extraction; rejected
    when content < 200 chars or the goal contains interactive verbs
    (click/fill/sort/filter/…), which forces escalation.
  - Layer 2a **deterministic**: replays caller-supplied
    `metadata.selectors = [{action, selector, value?}]` via Playwright.
  - Layer 2b **a11y**: text-only LLM loop over the accessibility-tree legend
    (V9 `/v1/chat`, gemini pin by default).
  - Layer 3 **vision**: set-of-marks screenshots to V9 `/v1/vision`.
  - **blocked**: CAPTCHA / Cloudflare / login-wall markers detected at every
    layer → `error_code="gateway_blocked"` → orchestrator recovery re-plans.
  - `metadata.force_path` escape hatch ('extract' | 'a11y' | 'vision').
- Engine integration: `skills.py` has a `browser` dispatch branch
  (sandbox_executor-style bypass of the LLM channel); `schemas.py` has
  `BrowserOutput` + `ErrorCode`; `agent_config.yaml` has the browser entry;
  `prompts/planner.md` knows when to schedule browser nodes and mandates a
  distiller between browser and formatter.
- Drivers already write per-turn evidence (`turn_NN_raw.png`,
  `turn_NN_marked.png`, `turn_NN_legend.txt`) under
  `state/sessions/<sid>/browser/browser_<ts>/<layer>/`.
- Terminal `replay.py` (kept as-is) and a pytest suite under `tests/`.

## 6. Work items

### 6.1 Gateway: port Anthropic V8 → V9

Mirror the user's existing V8 work: provider adapter class in
`providers.py`, register in `router.py` (LIMITS + SHORTCUTS) and `main.py`
(ORDER + TIER_TO_ORDER), copy `.env` keys from the V8 gateway, set
`agent_routing.yaml` pins (cognitive skills → anthropic, like S8). Default
model `claude-haiku-4-5-20251001`. The browser's internal layers keep their
defaults (a11y → gemini, vision → router-selected vision-capable provider).

### 6.2 Restore the user's S8 identity in the base

- Port `prompts/verifier.md` + its `agent_config.yaml` block from S8code,
  plus the planner/formatter prompt guidance lines that reference it.
- Replace the stub `prompts/coder.md` with the user's real S8 coder prompt.

### 6.3 Browser skill demo mode

In `browser/skill.py`, read `BROWSER_HEADFUL` and `BROWSER_SLOWMO_MS` env
vars and pass `headless=` / `slow_mo=` into both `chromium.launch()` calls
(`_drive` and `_try_deterministic`). Defaults unchanged (headless, no
slow-mo).

### 6.4 Demo task & cascade coverage

One submission run should naturally exercise multiple layers:

- Listing-page node: goal "filter models by the text-generation task, sort by
  most downloads, collect the top 3 model URLs" → interactive verbs →
  **a11y** layer, visible typing/clicking/sorting (≥3 interactions).
- One node demonstrates **deterministic** via `metadata.selectors` against
  Hugging Face's stable search box.
- Three model-detail nodes: pure extraction goals → **extract** layer.
- **vision** demonstrated via `metadata.force_path: vision` on one node in
  the video run (a separate run from the clean submission trace if needed).
- **blocked** demonstrated as a bonus run against a login-walled URL.

Planner prompt may need a small worked example for this DAG shape
(browser-list → 3× browser-detail → distiller → formatter, verifier on the
comparative claim). Keep changes prompt-only.

### 6.5 `replay_viewer.py` → `replay.html` (the 8 assignment items)

New standalone script; zero engine changes. Inputs: `state/sessions/<sid>/`
node records (skill, status, prompt_sent, AgentResult incl. BrowserOutput),
browser artifact folders, V9 cost/usage ledger (cost-by-agent endpoint or DB).
Output: one self-contained `replay.html` (all CSS inline, screenshots
base64-embedded) showing:

1. Original user goal
2. Planner DAG (diagram; dependency-free inline HTML/CSS layered rendering —
   no CDN scripts, so the file stays truly self-contained)
3. Browser path per browser node (cascade-layer badge:
   extract / deterministic / a11y / vision / blocked)
4. Exact actions taken (per-turn action list from BrowserOutput.actions)
5. Evidence (embedded per-turn screenshots + a11y legends)
6. Raw extracted data (BrowserOutput.content per node)
7. Final structured comparison table (formatter output)
8. Turn count + token/cost summary (per agent and total)

### 6.6 Deliverables & verification

- Run the base pytest suite after each engine-adjacent change.
- Clear memory before the submission run (S8 gotcha: memory pollution
  changes Planner routing).
- Capture the submission run into `traces/` (replay.html + raw JSON).
- Write README.md, RUN.md (PowerShell, two-terminal: gateway then agent),
  LEARNING_NOTES.md, ARCHITECTURE.md (the brief architecture note),
  YOUTUBE_SCRIPT.md.
- Push to a new GitHub repo.

## 7. Error handling

- `gateway_blocked` flows through the existing recovery path (re-plan); the
  replay viewer renders it as the "blocked" cascade layer, satisfying the
  assignment's layer taxonomy.
- Driver step failures: existing `max_failures=3` / `max_steps` budgets and
  90 s wall clock in `BrowserSkill` are kept.
- Hugging Face layout drift: goals are written against stable UI concepts
  (task filter, sort dropdown) rather than brittle selectors; the
  deterministic node's selectors are confined to the search box demo.

## 8. Teaching requirement

The user wants plain-English, step-by-step teaching during implementation:
each work item is explained (what / why) as it is built, jargon defined in
ordinary words.

## 9. Out of scope

- No edits to `flow.py`, `recovery.py`, `persistence.py`, S7 carryover files.
- No multi-site price comparison (rejected option), no e-commerce scraping.
- No live click-highlight injection (rejected option; slow-mo suffices).
- No changes to the terminal `replay.py`.
