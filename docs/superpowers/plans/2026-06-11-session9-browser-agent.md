# Session 9 Browser Agent Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the Session 9 browser agent (instructor base + user's S8 identity + Anthropic-enabled V9 gateway), demo it on a Hugging Face top-3 text-generation-model comparison, and emit a self-contained `replay.html` covering the 8 assignment trace items.

**Architecture:** The S8 growing-graph orchestrator runs unmodified; the browser skill (already integrated in the instructor base) drives Chromium through a 4-layer cascade (extract → deterministic → a11y → vision, blocked as structured failure). We add: an Anthropic provider port to the V9 gateway (with V9 multimodal translation), the user's S8 verifier/coder, env-var demo mode (headful/slow-mo), planner guidance, and a standalone `replay_viewer.py` that renders persisted session state + browser screenshots + gateway cost ledger into one HTML file.

**Tech Stack:** Python 3.11, uv, Playwright (Chromium), trafilatura, FastAPI gateway (V9, :8109), networkx, pytest. No agent frameworks (assignment rule). `flow.py` is never edited (assignment rule).

**Teaching requirement:** The user wants plain-English what/why explanations as each task is built. When executing, narrate each task: what the change is, why it's needed, and how it connects to the cascade/orchestrator mental model.

**Paths used throughout:**
- `ROOT` = `C:\The School Of AI\Session 9 - Browser Agents & Autonomous Web`
- `S8CODE` = `C:\The School Of AI\Session 8 - Multi-Agent DAG Orchestration (GRAPHS!)\S8code`
- `V8GW` = `C:\The School Of AI\Session 8 - Multi-Agent DAG Orchestration (GRAPHS!)\S8SharedCode\gateway`

---

## File Structure

```
ROOT\
├── S9code\                      ← NEW: copy of S9SharedCode\code (working copy)
│   ├── gateway.py               MODIFY: 1-line path fix (parents[2] → parents[1])
│   ├── agent_config.yaml        MODIFY: + verifier block; fix coder description
│   ├── prompts\verifier.md      NEW: copied from S8CODE
│   ├── prompts\coder.md         REPLACE: stub → S8CODE version
│   ├── prompts\planner.md       MODIFY: + verifier skill guidance (from S8)
│   ├── prompts\formatter.md     MODIFY: + verifier verdict bullet (from S8)
│   ├── browser\skill.py         MODIFY: env-var launch options (headful/slow-mo)
│   ├── replay_viewer.py         NEW: 8-item HTML replay generator
│   └── tests\
│       ├── test_browser_launch_options.py   NEW
│       └── test_replay_viewer.py            NEW
├── llm_gatewayV9\
│   ├── providers.py             MODIFY: + AnthropicProvider (ported V8 + V9 vision)
│   ├── router.py                MODIFY: + anthropic LIMITS + SHORTCUTS
│   ├── main.py                  MODIFY: + anthropic in DEFAULT_ORDER + TIER_TO_ORDER
│   ├── pricing.py               MODIFY: + anthropic price row
│   ├── agent_routing.yaml       MODIFY: S8-style pins (cognitive → anthropic)
│   └── .env                     NEW: copied from V8GW\.env
├── traces\                      NEW: committed submission run (replay.html + JSON)
└── README.md, RUN.md, ARCHITECTURE.md, LEARNING_NOTES.md, YOUTUBE_SCRIPT.md   NEW
```

---

### Task 0: Import the instructor base as S9code

**Files:** Create `ROOT\S9code\` (copy), modify `ROOT\S9code\gateway.py`.

- [ ] **Step 0.1: Copy the base, excluding junk and stale state**

```powershell
robocopy "$env:ROOT_S9\S9SharedCode\code" "$env:ROOT_S9\S9code" /E /XD __pycache__ .venv state\sessions /XF .DS_Store
# (set $env:ROOT_S9 = "C:\The School Of AI\Session 9 - Browser Agents & Autonomous Web" first)
```
Then reset memory so the instructor's cached facts don't steer our Planner:
```powershell
Set-Content "$env:ROOT_S9\S9code\state\memory.json" "[]" -Encoding utf8
Remove-Item "$env:ROOT_S9\S9code\state\index.faiss","$env:ROOT_S9\S9code\state\index_ids.json" -EA SilentlyContinue
```

- [ ] **Step 0.2: Fix the gateway path assumption**

`S9code\gateway.py:24` computes the gateway dir as `Path(__file__).resolve().parents[2] / "llm_gatewayV9"` — correct for `S9SharedCode\code\` (two levels deep) but wrong for `S9code\` (one level deep). Edit line 24:

```python
# old
GATEWAY_V9_DIR = Path(__file__).resolve().parents[2] / "llm_gatewayV9"
# new
GATEWAY_V9_DIR = Path(__file__).resolve().parents[1] / "llm_gatewayV9"
```

- [ ] **Step 0.3: Install deps + browser binary**

```powershell
cd "$env:ROOT_S9\S9code"
uv sync
uv run playwright install chromium
```
Expected: clean sync (pyproject already lists playwright/trafilatura/pillow), chromium downloaded.

- [ ] **Step 0.4: Agent-side .env**

`mcp_server.py` loads `S9code\.env` (Tavily key etc.). Copy the S8 agent env if present, else from the example:
```powershell
if (Test-Path "C:\The School Of AI\Session 8 - Multi-Agent DAG Orchestration (GRAPHS!)\S8code\.env") {
  Copy-Item "C:\The School Of AI\Session 8 - Multi-Agent DAG Orchestration (GRAPHS!)\S8code\.env" "$env:ROOT_S9\S9code\.env"
} else { Copy-Item "$env:ROOT_S9\S9code\.env.example" "$env:ROOT_S9\S9code\.env" }   # then fill keys
```

- [ ] **Step 0.5: Baseline test run**

```powershell
cd "$env:ROOT_S9\S9code"; uv run pytest -q
```
Expected: existing suite passes (tests marked `network`/`embed` may skip/fail without services — note which, that's the baseline).

- [ ] **Step 0.6: Commit**

```powershell
git -C $env:ROOT_S9 add S9code
git -C $env:ROOT_S9 commit -m "chore: import instructor S9 base as S9code working copy (gateway path fix for flat layout)"
```

---

### Task 1: Port the Anthropic provider into the V9 gateway

**Files:**
- Modify: `ROOT\llm_gatewayV9\providers.py` (after `GitHubProvider`, ~line 475)
- Modify: `ROOT\llm_gatewayV9\router.py:8-26`
- Modify: `ROOT\llm_gatewayV9\main.py:22,42-45`
- Modify: `ROOT\llm_gatewayV9\pricing.py:21-36`
- Modify: `ROOT\llm_gatewayV9\agent_routing.yaml`
- Create: `ROOT\llm_gatewayV9\.env`

**Why:** the browser's a11y/vision layers call this gateway; your cognitive skills route to Claude; the cost summary (deliverable #8) reads this gateway's ledger. V9's `/v1/vision` is required by the cascade's Layer 3.

- [ ] **Step 1.1: Copy the V8 provider class verbatim**

Copy `V8GW\providers.py` lines **353–592** (the `# Anthropic (Claude)` banner comment, `class AnthropicProvider`, and `_anthropic_clean_schema`) into `ROOT\llm_gatewayV9\providers.py`, inserting after `class GitHubProvider` (~line 475). `_flatten_system` and `_gemini_inline_refs` already exist in V9 (lines 127 and 710 — `_anthropic_clean_schema` references the latter, defined later in the file; fine at call time).

- [ ] **Step 1.2: Ensure imports**

At the top of V9 `providers.py`, add (if absent — `httpx` is there, `uuid`/`json` are not):
```python
import json
import uuid
```

- [ ] **Step 1.3: V9 adaptation — declare vision + translate image blocks**

V9 auto-bakes capabilities from `VISION_MODEL_HINTS` (claude models match), so the router WILL send screenshots to Anthropic — but the V8 `_translate_messages` would `json.dumps` them. Two edits inside the pasted class:

(a) capabilities dict — add the vision key:
```python
    capabilities = {
        "tools": True, "caching": True, "reasoning": True,
        "structured": True, "parallel_tools": True, "vision": True,
    }
```

(b) in `_translate_messages`, replace the final fallthrough block:
```python
            content = m.get("content", "")
            out.append({
                "role": r if r in ("user", "assistant") else "user",
                "content": content if isinstance(content, str) else json.dumps(content),
            })
```
with:
```python
            content = m.get("content", "")
            # V9 multimodal: canonical image_url blocks (pre-resolved to data:
            # URLs by main._resolve_image_urls) become Anthropic image blocks.
            if isinstance(content, list) and _content_has_image(content):
                blocks = []
                for media_type, b64 in _iter_image_blocks(content):
                    blocks.append({
                        "type": "image",
                        "source": {"type": "base64",
                                   "media_type": media_type, "data": b64},
                    })
                text = _extract_text_blocks(content)
                if text:
                    blocks.append({"type": "text", "text": text})
                out.append({
                    "role": r if r in ("user", "assistant") else "user",
                    "content": blocks,
                })
                continue
            out.append({
                "role": r if r in ("user", "assistant") else "user",
                "content": content if isinstance(content, str) else json.dumps(content),
            })
```
(`_content_has_image`, `_iter_image_blocks`, `_extract_text_blocks` are module-level helpers at providers.py:60-113.)

- [ ] **Step 1.4: Register in the worker pool**

In `build_providers` (providers.py:963), add as the FIRST entry of `out`:
```python
    # Anthropic — paid tier, generous limits; default model Claude Haiku 4.5
    # ($1/$5 per MTok, vision-capable).
    if k := os.getenv("ANTHROPIC_API_KEY"):
        out["anthropic"] = AnthropicProvider(
            k, os.getenv("ANTHROPIC_MODEL", "claude-haiku-4-5-20251001")
        )
```
The existing capability-bake loop (lines 988-989) runs after and will confirm `vision: True` via the model hints.

- [ ] **Step 1.5: router.py — LIMITS + SHORTCUTS**

In `LIMITS` (router.py:8) add as first entry:
```python
    "anthropic":  {"rpm": 100,  "rpd": 9999,    "tpm": 200000,   "cooldown": 1,   "max_ctx": 200000},
```
In `SHORTCUTS` (router.py:18) add as first line:
```python
    "a": "anthropic", "an": "anthropic", "anthropic": "anthropic",
    "cl": "anthropic", "claude": "anthropic",
```

- [ ] **Step 1.6: main.py — ORDER + TIER_TO_ORDER**

Line 22:
```python
DEFAULT_ORDER = ["anthropic", "ollama", "gemini", "nvidia", "groq", "cerebras", "openrouter", "github"]
```
Lines 42-45:
```python
TIER_TO_ORDER = {
    "TINY":  ["anthropic", "github", "openrouter", "groq", "nvidia", "cerebras", "gemini", "ollama"],
    "LARGE": ["anthropic", "gemini", "groq", "nvidia", "cerebras", "github", "openrouter", "ollama"],
}
```

- [ ] **Step 1.7: pricing.py — Claude Haiku 4.5 row**

In `PRICING_USD_PER_MTOK` add:
```python
    # Anthropic — Claude Haiku 4.5 list rate as of 2026-06 ($1 in / $5 out per MTok).
    "anthropic":  (1.00, 5.00),
```

- [ ] **Step 1.8: agent_routing.yaml — S8-style pins**

Replace the pin lines (keep the header comment) with:
```yaml
# Session 9 routing: cognitive skills → Anthropic (Claude Haiku 4.5); Critic →
# Groq for provider diversity in traces; browser → gemini (the a11y layer
# pins gemini in code; the vision layer needs a vision-capable provider and
# gemini's free tier handles screenshots).
planner: anthropic
researcher: anthropic
distiller: anthropic
summariser: anthropic
critic: groq
formatter: anthropic
retriever: anthropic
sandbox_executor: anthropic   # never hits the LLM (bypassed); pin is cosmetic
coder: anthropic
verifier: anthropic
browser: gemini
```

- [ ] **Step 1.9: Create .env**

```powershell
Copy-Item "C:\The School Of AI\Session 8 - Multi-Agent DAG Orchestration (GRAPHS!)\S8SharedCode\gateway\.env" "$env:ROOT_S9\llm_gatewayV9\.env"
```
Then open it and confirm `ANTHROPIC_API_KEY` and `ANTHROPIC_MODEL=claude-haiku-4-5-20251001` are present; remove any `GATEWAY_V8_PORT` line (V9 reads `GATEWAY_V9_PORT`, default 8109 is fine).

- [ ] **Step 1.10: Verify — boot + chat + vision**

Terminal 1: `cd "$env:ROOT_S9\llm_gatewayV9"; uv sync; uv run main.py` → expect startup log listing anthropic among providers, port 8109.
Terminal 2 (chat through the gateway's own client):
```powershell
cd "$env:ROOT_S9\llm_gatewayV9"
uv run python -c "from client import LLM; r = LLM().chat(prompt='Reply with exactly: OK', provider='a'); print(r['provider'], r['text'])"
```
Expected: `anthropic OK`.
Vision smoke (exercises Step 1.3's translation — first Read `schemas.py`'s `VisionRequest` to confirm field names, then):
```powershell
uv run python -c "import httpx, json; img='iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mP8z8BQDwAEhQGAhKmMIQAAAABJRU5ErkJggg=='; r = httpx.post('http://localhost:8109/v1/vision', json={'image': 'data:image/png;base64,'+img, 'prompt': 'One word: what colour is this pixel?', 'provider': 'a'}, timeout=60); print(r.status_code, r.json())"
```
Expected: 200 and a reply mentioning red (adjust body keys to the actual `VisionRequest` schema if 422).

- [ ] **Step 1.11: Commit**

```powershell
git -C $env:ROOT_S9 add llm_gatewayV9
git -C $env:ROOT_S9 commit -m "feat(gateway): port Anthropic provider to V9 with multimodal translation, routing pins, pricing"
```
(Note: `.env` is gitignored — verify `git status` does not list it.)

---

### Task 2: Restore the S8 identity (verifier + coder)

**Files:**
- Create: `ROOT\S9code\prompts\verifier.md` (copy of `S8CODE\prompts\verifier.md`)
- Replace: `ROOT\S9code\prompts\coder.md` (copy of `S8CODE\prompts\coder.md`)
- Modify: `ROOT\S9code\agent_config.yaml`, `prompts\planner.md`, `prompts\formatter.md`

- [ ] **Step 2.1: Copy the two prompt files**

```powershell
Copy-Item "$env:S8CODE\prompts\verifier.md" "$env:ROOT_S9\S9code\prompts\verifier.md"
Copy-Item "$env:S8CODE\prompts\coder.md" "$env:ROOT_S9\S9code\prompts\coder.md" -Force
```

- [ ] **Step 2.2: agent_config.yaml — add verifier, fix coder description**

Append after the `verifier`-less catalogue (before or after `browser:` — order is cosmetic):
```yaml
verifier:
  prompt: prompts/verifier.md
  tools_allowed: []
  temperature: 0.0         # checking a claim should be deterministic
  max_tokens: 600
  description: Independently checks whether a specific claim is supported by the evidence in its inputs; emits holds/refuted/unsupported with a confidence score. Unlike critic, it annotates rather than triggering re-planning.
```
And replace the coder entry's `description: STUB. Student assignment for Session 8...` line with:
```yaml
  description: Writes a short Python program to compute an exact answer; the orchestrator runs it in the sandbox and a formatter presents the executed result.
```

- [ ] **Step 2.3: planner.md — teach the Planner the verifier exists**

(a) In the `Available skills:` list at the top of `S9code\prompts\planner.md`, add one line (alongside the other skills):
```
  verifier           independently checks a claim against evidence (confidence-scored)
```
(b) Append this paragraph (verbatim from S8 planner.md:70-79) after the coder auto-format paragraph:
```
When the user asks to verify / fact-check / double-check a claim, or when
the correctness of one specific factual or numeric claim is the whole
point of the request, add a `verifier` node. Wire the evidence nodes
(researcher / retriever / coder results) as its `inputs`, and put the exact
claim to check in its metadata.question. Then wire the final `formatter` to
depend on BOTH the evidence and the verifier, so it can report the verdict
and add an honest caveat when the claim does not hold. A `verifier` is NOT
a `critic`: it does not trigger re-planning, it annotates the answer. (If a
`coder` is part of this graph, remember the formatter is automatic — in
that case put the verifier BEFORE the coder, checking the raw evidence.)
```
(If S9's planner.md lacks a "coder auto-formats" paragraph, append at the end of the guidance section — placement is not load-bearing.)

- [ ] **Step 2.4: formatter.md — verdict bullet**

Append to the rules list in `S9code\prompts\formatter.md` (verbatim from S8 formatter.md:32-35):
```
  - If an upstream `verifier` node is present, respect its verdict: when it
    is "holds", you may state the claim confidently; when it is "refuted" or
    "unsupported", add a short honest caveat (e.g. "note: this could not be
    confirmed from the evidence") and reflect its `reason`.
```

- [ ] **Step 2.5: Verify registry loads**

```powershell
cd "$env:ROOT_S9\S9code"
uv run python -c "from skills import SkillRegistry; n = SkillRegistry().names(); print(n); assert 'verifier' in n and 'browser' in n"
uv run pytest -q
```
Expected: verifier listed; suite matches Task 0 baseline.

- [ ] **Step 2.6: Commit**

```powershell
git -C $env:ROOT_S9 add S9code
git -C $env:ROOT_S9 commit -m "feat(agent): restore S8 verifier skill and real coder prompt in S9 catalogue"
```

---

### Task 3: Browser demo mode (headful + slow-mo via env vars) — TDD

**Files:**
- Modify: `ROOT\S9code\browser\skill.py`
- Test: `ROOT\S9code\tests\test_browser_launch_options.py`

- [ ] **Step 3.1: Write the failing test**

```python
"""Demo-mode launch options: BROWSER_HEADFUL / BROWSER_SLOWMO_MS env vars."""
from browser.skill import _launch_options


def test_default_is_headless(monkeypatch):
    monkeypatch.delenv("BROWSER_HEADFUL", raising=False)
    monkeypatch.delenv("BROWSER_SLOWMO_MS", raising=False)
    assert _launch_options() == {"headless": True}


def test_headful_with_slowmo(monkeypatch):
    monkeypatch.setenv("BROWSER_HEADFUL", "1")
    monkeypatch.setenv("BROWSER_SLOWMO_MS", "400")
    assert _launch_options() == {"headless": False, "slow_mo": 400}


def test_garbage_slowmo_ignored(monkeypatch):
    monkeypatch.setenv("BROWSER_HEADFUL", "0")
    monkeypatch.setenv("BROWSER_SLOWMO_MS", "fast")
    assert _launch_options() == {"headless": True}
```

- [ ] **Step 3.2: Run, verify fail**

`uv run pytest tests/test_browser_launch_options.py -v` → expect `ImportError: cannot import name '_launch_options'`.

- [ ] **Step 3.3: Implement**

In `S9code\browser\skill.py`: add `import os` to the imports, and below the `_UA` constant add:
```python
def _launch_options() -> dict:
    """Demo-mode Chromium options from env. BROWSER_HEADFUL=1 opens a visible
    browser window; BROWSER_SLOWMO_MS=<int> pauses that many ms between
    Playwright actions so a human (or a screen recording) can follow the
    clicks. Defaults preserve normal headless operation."""
    opts: dict = {"headless": os.environ.get("BROWSER_HEADFUL") != "1"}
    slow = os.environ.get("BROWSER_SLOWMO_MS", "")
    if slow.isdigit() and int(slow) > 0:
        opts["slow_mo"] = int(slow)
    return opts
```
Then replace BOTH launch sites (`_drive` at ~line 257 and `_try_deterministic` at ~line 315):
```python
            browser = await p.chromium.launch(headless=True)
```
→
```python
            browser = await p.chromium.launch(**_launch_options())
```

- [ ] **Step 3.4: Run, verify pass** — `uv run pytest tests/test_browser_launch_options.py -v` → 3 passed. Then full `uv run pytest -q`.

- [ ] **Step 3.5: Commit**

```powershell
git -C $env:ROOT_S9 add S9code/browser/skill.py S9code/tests/test_browser_launch_options.py
git -C $env:ROOT_S9 commit -m "feat(browser): BROWSER_HEADFUL/BROWSER_SLOWMO_MS env vars for visible demo runs"
```

---

### Task 4: Planner worked example for the comparison DAG

**Files:** Modify `ROOT\S9code\prompts\planner.md`.

The base planner already teaches when to use browser (with HF examples). Add one compact worked example so the seed plan reliably takes the shape *list-page browser → 3 detail browsers → distiller → formatter (+ verifier)*.

- [ ] **Step 4.1: Append the example**

Add at the end of planner.md's browser guidance:

```
Worked example — "compare the top 3 text-generation models on Hugging Face
by downloads": emit ONE browser node (url=https://huggingface.co/models,
goal="filter Tasks=Text Generation, sort by Most Downloads, extract the top
3 model names, their URLs, downloads and likes"). Let that node's OUTPUT
drive the rest: emit a distiller depending on it to pull the 3 model URLs
into structured fields, then (as successors from the distiller or in your
recovery turn) one browser node per model page with an extraction goal
("extract downloads, likes, license, parameter count"), a distiller over
those three, and a formatter that depends on the distiller to render the
comparison table. If the user's request hinges on the ranking being right,
add a verifier between the distiller and formatter checking "these are the
top 3 by downloads" against the list-page evidence.
```

- [ ] **Step 4.2: Sanity check + commit**

`uv run python -c "from skills import SkillRegistry; SkillRegistry()"` (prompt files are read lazily; this just guards YAML). Commit:
```powershell
git -C $env:ROOT_S9 add S9code/prompts/planner.md
git -C $env:ROOT_S9 commit -m "feat(planner): worked example for HF top-3 comparison DAG shape"
```

---

### Task 5: `replay_viewer.py` — the 8-item HTML report (TDD)

**Files:**
- Create: `ROOT\S9code\replay_viewer.py`
- Test: `ROOT\S9code\tests\test_replay_viewer.py`

**Data sources (all existing):** `state/sessions/<sid>/query.txt`, `graph.json` (nx node_link with `edges`), `nodes/n_*.json` (NodeState: skill, status, prompt_sent, result.output incl. BrowserOutput `path`/`actions`/`content`/`turns`, result.error_code), browser artifacts `state/sessions/<sid>/browser/browser_<ts>/<layer>/turn_NN_{raw.png,marked.png,legend.txt}`, and `GET :8109/v1/cost/by_agent?session=<sid>`.

- [ ] **Step 5.1: Write the failing test**

```python
"""replay_viewer renders all 8 assignment items into one self-contained HTML."""
import base64
import json
from pathlib import Path

import persistence
import replay_viewer

# 1x1 red PNG
_PNG = base64.b64decode(
    "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mP8z8BQDwAEhQGAhKmMIQAAAABJRU5ErkJggg=="
)


def _write_session(root: Path, sid: str) -> None:
    d = root / sid
    (d / "nodes").mkdir(parents=True)
    (d / "query.txt").write_text("compare top 3 HF models", encoding="utf-8")
    (d / "graph.json").write_text(json.dumps({
        "directed": True, "multigraph": False, "graph": {},
        "nodes": [{"id": "n:1"}, {"id": "n:2"}],
        "edges": [{"source": "n:1", "target": "n:2"}],
    }), encoding="utf-8")
    (d / "nodes" / "n_001.json").write_text(json.dumps({
        "node_id": "n:1", "skill": "browser", "status": "complete",
        "inputs": [], "prompt_sent": "p1", "retries": 0,
        "result": {"success": True, "agent_name": "browser",
                   "output": {"url": "https://huggingface.co/models",
                              "goal": "filter and sort", "path": "a11y",
                              "turns": 4,
                              "content": "RAW-EXTRACTED-MODEL-LIST",
                              "actions": [{"turn": 1,
                                           "actions": [{"kind": "click", "target": "Sort"}],
                                           "outcome": "ok"}],
                              "final_url": "https://huggingface.co/models?sort=downloads"},
                   "successors": [], "cost": 0.0, "elapsed_s": 3.2,
                   "provider": "gemini", "error": None, "error_code": None},
    }), encoding="utf-8")
    (d / "nodes" / "n_002.json").write_text(json.dumps({
        "node_id": "n:2", "skill": "formatter", "status": "complete",
        "inputs": ["n:1"], "prompt_sent": "p2", "retries": 0,
        "result": {"success": True, "agent_name": "formatter",
                   "output": {"final_answer": "| model | downloads |\n|---|---|"},
                   "successors": [], "cost": 0.0, "elapsed_s": 1.0,
                   "provider": "anthropic", "error": None, "error_code": None},
    }), encoding="utf-8")
    ev = d / "browser" / "browser_123" / "a11y"
    ev.mkdir(parents=True)
    (ev / "turn_01_raw.png").write_bytes(_PNG)
    (ev / "turn_01_legend.txt").write_text("[1]<button>Sort</button>", encoding="utf-8")


def test_renders_all_eight_items(tmp_path, monkeypatch):
    monkeypatch.setattr(persistence, "SESSIONS_ROOT", tmp_path)
    monkeypatch.setattr(replay_viewer, "SESSIONS_ROOT", tmp_path)
    _write_session(tmp_path, "s1")
    html = replay_viewer.render_session("s1", gateway_url="http://localhost:1")  # port 1 → offline
    assert "compare top 3 HF models" in html                  # 1 goal
    assert "n:1" in html and "n:2" in html and "&rarr;" in html  # 2 DAG nodes+edge
    assert "a11y" in html                                     # 3 cascade layer badge
    assert "click" in html and "Sort" in html                 # 4 exact actions
    assert "base64," in html and "&lt;button&gt;" in html     # 5 evidence (img + legend, escaped)
    assert "RAW-EXTRACTED-MODEL-LIST" in html                 # 6 raw data
    assert "| model | downloads |" in html                    # 7 final table
    assert "turns" in html.lower() and "gateway offline" in html.lower()  # 8 turns + cost fallback


def test_writes_file(tmp_path, monkeypatch):
    monkeypatch.setattr(persistence, "SESSIONS_ROOT", tmp_path)
    monkeypatch.setattr(replay_viewer, "SESSIONS_ROOT", tmp_path)
    _write_session(tmp_path, "s2")
    out = replay_viewer.write_report("s2", gateway_url="http://localhost:1")
    assert out.exists() and out.name == "replay.html"
```

- [ ] **Step 5.2: Run, verify fail** — `uv run pytest tests/test_replay_viewer.py -v` → `ModuleNotFoundError: replay_viewer`.

- [ ] **Step 5.3: Implement `S9code\replay_viewer.py`**

```python
"""Session 9 replay viewer — renders one persisted run as a single
self-contained replay.html covering the 8 assignment trace items:

  1 user goal · 2 planner DAG · 3 browser cascade layer · 4 exact actions
  5 evidence (screenshots + a11y legends) · 6 raw extracted data
  7 final comparison table · 8 turn count + token/cost summary

Standalone: reads state/sessions/<sid>/ (via persistence.SessionStore),
the browser artifact folders, and the V9 gateway cost ledger. Zero engine
changes; no CDN scripts; screenshots are base64-embedded.

Usage:  uv run python replay_viewer.py <session_id> [gateway_url]
"""
from __future__ import annotations

import base64
import html as html_mod
import json
import sys
from pathlib import Path

import httpx

from persistence import SESSIONS_ROOT, SessionStore

DEFAULT_GATEWAY = "http://localhost:8109"

_CSS = """
body{font-family:Segoe UI,system-ui,sans-serif;margin:24px;background:#fafafa;color:#222}
h1{font-size:1.4em} h2{border-bottom:2px solid #ddd;padding-bottom:4px;margin-top:32px}
.badge{display:inline-block;padding:2px 10px;border-radius:10px;font-size:.85em;color:#fff}
.badge.extract{background:#2e7d32}.badge.deterministic{background:#1565c0}
.badge.a11y{background:#ef6c00}.badge.vision{background:#8e24aa}.badge.blocked{background:#c62828}
.node{background:#fff;border:1px solid #ddd;border-radius:8px;padding:10px;margin:6px;display:inline-block;vertical-align:top;min-width:160px}
.level{margin:4px 0} .edge{color:#888;font-size:.9em}
table{border-collapse:collapse;background:#fff}td,th{border:1px solid #ccc;padding:4px 10px}
img.shot{max-width:480px;border:1px solid #bbb;margin:4px}
pre{background:#272822;color:#f8f8f2;padding:10px;border-radius:6px;overflow-x:auto;max-height:320px}
details>summary{cursor:pointer;font-weight:600;margin:6px 0}
"""


def _esc(s) -> str:
    return html_mod.escape(str(s if s is not None else ""))


# ── data loading ─────────────────────────────────────────────────────────────

def _load_edges(store: SessionStore) -> list[tuple[str, str]]:
    if not store.graph_path.exists():
        return []
    payload = json.loads(store.graph_path.read_text(encoding="utf-8"))
    return [(e["source"], e["target"]) for e in payload.get("edges", [])]


def _dag_levels(node_ids: list[str], edges: list[tuple[str, str]]) -> list[list[str]]:
    """Longest-path layering — good enough for a small DAG, no deps needed."""
    preds: dict[str, set[str]] = {n: set() for n in node_ids}
    for u, v in edges:
        preds.setdefault(v, set()).add(u)
        preds.setdefault(u, set())
    level: dict[str, int] = {}

    def depth(n: str, seen=()) -> int:
        if n in level:
            return level[n]
        if n in seen:                       # cycle guard; DAGs shouldn't have any
            return 0
        d = 0 if not preds[n] else 1 + max(depth(p, seen + (n,)) for p in preds[n])
        level[n] = d
        return d

    for n in preds:
        depth(n)
    out: list[list[str]] = [[] for _ in range(max(level.values(), default=0) + 1)]
    for n, d in sorted(level.items()):
        out[d].append(n)
    return out


def _evidence_files(session_dir: Path) -> list[Path]:
    root = session_dir / "browser"
    return sorted(root.rglob("turn_*")) if root.exists() else []


def _fetch_costs(session_id: str, gateway_url: str):
    try:
        r = httpx.get(f"{gateway_url}/v1/cost/by_agent",
                      params={"session": session_id}, timeout=5)
        r.raise_for_status()
        return r.json()
    except Exception:
        return None


# ── rendering ────────────────────────────────────────────────────────────────

def _browser_path(node) -> str:
    r = node.result
    if r is None:
        return "?"
    if r.error_code == "gateway_blocked":
        return "blocked"
    return (r.output or {}).get("path", "?")


def _render_dag(states, edges) -> str:
    by_id = {s.node_id: s for s in states}
    ids = sorted(set(by_id) | {n for e in edges for n in e})
    rows = []
    for lvl in _dag_levels(ids, edges):
        cells = []
        for n in lvl:
            st = by_id.get(n)
            label = f"<b>{_esc(n)}</b><br>{_esc(st.skill) if st else '?'}"
            if st and st.skill == "browser":
                p = _browser_path(st)
                label += f'<br><span class="badge {p}">{_esc(p)}</span>'
            status = _esc(st.status) if st else "?"
            cells.append(f'<span class="node">{label}<br><small>{status}</small></span>')
        rows.append(f'<div class="level">{"".join(cells)}</div>')
    edge_txt = " &nbsp; ".join(f'<span class="edge">{_esc(u)} &rarr; {_esc(v)}</span>'
                               for u, v in edges)
    return "".join(rows) + f"<p>{edge_txt}</p>"


def _render_browser_nodes(states) -> tuple[str, str, int]:
    """Returns (actions_html, raw_data_html, total_browser_turns)."""
    actions, raw, turns = [], [], 0
    for st in states:
        if st.skill != "browser" or st.result is None:
            continue
        out = st.result.output or {}
        turns += int(out.get("turns") or 0)
        path = _browser_path(st)
        actions.append(
            f"<h3>{_esc(st.node_id)} — <span class='badge {path}'>{_esc(path)}</span> "
            f"&nbsp;<small>{_esc(out.get('url'))} → {_esc(out.get('final_url'))}</small></h3>"
            f"<p><i>goal:</i> {_esc(out.get('goal'))}</p>"
            f"<pre>{_esc(json.dumps(out.get('actions') or [], indent=2))}</pre>"
        )
        if st.result.error:
            actions.append(f"<p><b>error:</b> {_esc(st.result.error)}</p>")
        content = out.get("content")
        if content:
            raw.append(f"<details><summary>{_esc(st.node_id)} raw extract "
                       f"({len(content)} chars)</summary><pre>{_esc(content)}</pre></details>")
    return "".join(actions) or "<p>(no browser nodes)</p>", \
           "".join(raw) or "<p>(no extracted content)</p>", turns


def _render_evidence(files: list[Path]) -> str:
    parts = []
    for f in files:
        rel = _esc("/".join(f.parts[-3:]))
        if f.suffix == ".png":
            b64 = base64.b64encode(f.read_bytes()).decode()
            parts.append(f'<figure><img class="shot" src="data:image/png;base64,{b64}">'
                         f"<figcaption>{rel}</figcaption></figure>")
        elif f.suffix == ".txt":
            parts.append(f"<details><summary>{rel}</summary>"
                         f"<pre>{_esc(f.read_text(encoding='utf-8', errors='replace'))}</pre></details>")
    return "".join(parts) or "<p>(no browser artifacts captured)</p>"


def _render_costs(costs, states, browser_turns: int) -> str:
    n_nodes = len(states)
    head = (f"<p><b>Graph nodes run:</b> {n_nodes} &nbsp; "
            f"<b>Browser turns (LLM-driven steps):</b> {browser_turns}</p>")
    if costs is None:
        return head + "<p><i>Cost ledger unavailable (gateway offline when this report was generated).</i></p>"
    rows, t_in, t_out, t_usd = [], 0, 0, 0.0
    for agent, entries in sorted(costs.items()):
        for e in entries if isinstance(entries, list) else [entries]:
            i = int(e.get("input_tokens", e.get("in", 0)) or 0)
            o = int(e.get("output_tokens", e.get("out", 0)) or 0)
            d = float(e.get("dollars", 0) or 0)
            prov = _esc(e.get("provider", ""))
            t_in, t_out, t_usd = t_in + i, t_out + o, t_usd + d
            rows.append(f"<tr><td>{_esc(agent)}</td><td>{prov}</td>"
                        f"<td>{i}</td><td>{o}</td><td>${d:.6f}</td></tr>")
    return (head + "<table><tr><th>agent</th><th>provider</th><th>in tokens</th>"
            "<th>out tokens</th><th>$ est.</th></tr>" + "".join(rows) +
            f"<tr><th>total</th><th></th><th>{t_in}</th><th>{t_out}</th>"
            f"<th>${t_usd:.6f}</th></tr></table>")


def render_session(session_id: str, gateway_url: str = DEFAULT_GATEWAY) -> str:
    store = SessionStore(session_id)
    states = store.read_all_nodes()
    query = store.read_query()
    edges = _load_edges(store)
    actions_html, raw_html, browser_turns = _render_browser_nodes(states)
    final = next((s.result.output.get("final_answer")
                  for s in reversed(states)
                  if s.skill == "formatter" and s.result and s.result.output
                  and s.result.output.get("final_answer")), None)
    costs = _fetch_costs(session_id, gateway_url)
    paths = sorted({_browser_path(s) for s in states if s.skill == "browser"})
    sections = [
        f"<h1>Replay — session {_esc(session_id)}</h1>",
        f"<h2>1 · User goal</h2><blockquote>{_esc(query)}</blockquote>",
        f"<h2>2 · Planner DAG</h2>{_render_dag(states, edges)}",
        f"<h2>3 · Browser cascade path</h2><p>Layers used: "
        + " ".join(f'<span class="badge {p}">{_esc(p)}</span>' for p in paths or ["?"]) + "</p>",
        f"<h2>4 · Exact browser actions</h2>{actions_html}",
        f"<h2>5 · Evidence (per-turn screenshots &amp; a11y legends)</h2>"
        + _render_evidence(_evidence_files(store.dir)),
        f"<h2>6 · Raw extracted data</h2>{raw_html}",
        f"<h2>7 · Final comparison table</h2><pre>{_esc(final or '(no formatter output)')}</pre>",
        f"<h2>8 · Turns &amp; cost summary</h2>{_render_costs(costs, states, browser_turns)}",
    ]
    return ("<!doctype html><html><head><meta charset='utf-8'>"
            f"<title>replay {_esc(session_id)}</title><style>{_CSS}</style></head>"
            f"<body>{''.join(sections)}</body></html>")


def write_report(session_id: str, gateway_url: str = DEFAULT_GATEWAY) -> Path:
    out = SessionStore(session_id).dir / "replay.html"
    out.write_text(render_session(session_id, gateway_url), encoding="utf-8")
    return out


if __name__ == "__main__":
    if len(sys.argv) < 2:
        sessions = sorted(p.name for p in SESSIONS_ROOT.iterdir()) if SESSIONS_ROOT.exists() else []
        print("usage: python replay_viewer.py <session_id> [gateway_url]")
        print("sessions:", *sessions[-10:], sep="\n  ")
        raise SystemExit(2)
    gw = sys.argv[2] if len(sys.argv) > 2 else DEFAULT_GATEWAY
    path = write_report(sys.argv[1], gw)
    print(f"wrote {path}")
```

NOTE for the implementer: the `SessionStore` instance uses `persistence.SESSIONS_ROOT` captured at init, so the test monkeypatches `persistence.SESSIONS_ROOT`. If `SessionStore.__init__` resolves the module global at call time (it does — `SESSIONS_ROOT / session_id`), monkeypatching `persistence.SESSIONS_ROOT` is sufficient; the extra `replay_viewer.SESSIONS_ROOT` patch covers the `__main__` listing only. If the cost-row shape from `/v1/cost/by_agent` differs from the keys guessed in `_render_costs` (check `llm_gatewayV9/main.py:671-700` and `db.by_agent`), adjust the key names there — the test only exercises the offline fallback.

- [ ] **Step 5.4: Run, verify pass** — `uv run pytest tests/test_replay_viewer.py -v` → 2 passed; then full suite.

- [ ] **Step 5.5: Commit**

```powershell
git -C $env:ROOT_S9 add S9code/replay_viewer.py S9code/tests/test_replay_viewer.py
git -C $env:ROOT_S9 commit -m "feat: replay_viewer.py renders 8-item self-contained replay.html per session"
```

---

### Task 6: End-to-end demo run + traces

**Files:** Create `ROOT\traces\` (run outputs committed).

- [ ] **Step 6.1: Start the gateway** (terminal 1)

```powershell
cd "$env:ROOT_S9\llm_gatewayV9"; uv run main.py
```

- [ ] **Step 6.2: Clean memory, set demo env, run** (terminal 2)

```powershell
cd "$env:ROOT_S9\S9code"
Set-Content .\state\memory.json "[]" -Encoding utf8
Remove-Item .\state\index.faiss, .\state\index_ids.json -EA SilentlyContinue
$env:BROWSER_HEADFUL = "1"; $env:BROWSER_SLOWMO_MS = "400"; $env:PYTHONUNBUFFERED = "1"
uv run python flow.py "Compare the top 3 text-generation models on Hugging Face sorted by downloads - report name, downloads, likes, license and parameter count in a comparison table"
```
Expected: visible Chromium opens on huggingface.co/models, applies the Text Generation filter, sorts by downloads, model pages get visited; run ends with a formatter table. Note the printed session id. **If the listing node lands on `extract` instead of `a11y`** (no visible interaction), the goal phrasing lacked interactive verbs — re-run after checking the planner emitted a goal containing "filter/sort" (the `_is_useful_extract` gate at `browser/skill.py:106` keys on those verbs).

- [ ] **Step 6.3: Generate the replay**

```powershell
uv run python replay_viewer.py <session-id>
Start-Process .\state\sessions\<session-id>\replay.html
```
Manually verify all 8 sections are populated (cost table must show real tokens — gateway is up).

- [ ] **Step 6.4: Bonus runs for the video (optional but planned)**
  - Vision layer: re-run the same query after adding `force_path: vision` via a one-off direct node run, or simply run `flow.py` with a goal naming a JS-canvas-heavy page; simplest: temporarily set `metadata.force_path` in a small driver script. Keep whatever evidence lands in artifacts.
  - Blocked layer: `uv run python flow.py "From https://www.instagram.com/accounts/login extract the trending posts list"` → expect `gateway_blocked` and a recovery re-plan in the trace.

- [ ] **Step 6.5: Copy the submission trace + commit**

```powershell
New-Item -ItemType Directory -Force "$env:ROOT_S9\traces\<session-id>"
Copy-Item "$env:ROOT_S9\S9code\state\sessions\<session-id>\replay.html" "$env:ROOT_S9\traces\<session-id>\"
Copy-Item "$env:ROOT_S9\S9code\state\sessions\<session-id>\graph.json" "$env:ROOT_S9\traces\<session-id>\"
Copy-Item "$env:ROOT_S9\S9code\state\sessions\<session-id>\query.txt" "$env:ROOT_S9\traces\<session-id>\"
Copy-Item -Recurse "$env:ROOT_S9\S9code\state\sessions\<session-id>\nodes" "$env:ROOT_S9\traces\<session-id>\nodes"
git -C $env:ROOT_S9 add traces; git -C $env:ROOT_S9 commit -m "docs: submission trace - HF top-3 comparison run (replay.html + raw session state)"
```
(`state/sessions/` is gitignored in S9code; `traces\` is the curated committed copy.)

---

### Task 7: Documentation + publish

**Files:** Create `ROOT\README.md`, `RUN.md`, `ARCHITECTURE.md`, `LEARNING_NOTES.md`, `YOUTUBE_SCRIPT.md`.

- [ ] **Step 7.1: README.md** — what the project is, the cascade diagram (extract → deterministic → a11y → vision / blocked), the demo task, deliverable map (assignment item → where it lives), repo layout, link to traces/.
- [ ] **Step 7.2: RUN.md** — the exact two-terminal PowerShell flow from Task 6 (gateway, env vars, flow.py, replay_viewer.py), prereqs (uv, Python 3.11+, `playwright install chromium`, .env keys), and the memory-reset gotcha.
- [ ] **Step 7.3: ARCHITECTURE.md** (the assignment's "brief architecture note") — how the browser skill plugs in via the skills.py dispatch (sandbox_executor pattern), the cascade's escalation/blocked rules, what was built vs ported (Anthropic V9 port, demo mode, replay viewer, verifier/coder restore), and the no-flow.py-edits invariant.
- [ ] **Step 7.4: LEARNING_NOTES.md** — plain-English teaching recap (cost-ladder rationale, a11y tree vs set-of-marks, gateway_blocked as a first-class outcome, why the engine never changed).
- [ ] **Step 7.5: YOUTUBE_SCRIPT.md** — scene-by-scene: goal → DAG appears → headful browser filters/sorts/clicks (3+ visible actions) → replay.html walkthrough of the 8 items → cost table.
- [ ] **Step 7.6: Final verification** — run `uv run pytest -q` once more; confirm `git status` clean except intended files; confirm replay.html in traces opens standalone (no network needed).
- [ ] **Step 7.7: Publish** — create the GitHub repo (user account choice), `git remote add origin <url>`, push master. The user records the video separately using RUN.md + YOUTUBE_SCRIPT.md.

---

## Self-review notes

- **Spec coverage:** §6.1 → Task 1; §6.2 → Task 2; §6.3 → Task 3; §6.4 → Tasks 4+6; §6.5 → Task 5; §6.6 → Tasks 6+7; repo layout → Task 0. Cascade-layer coverage (extract/deterministic/a11y/vision/blocked) is exercised across Task 6 runs; deterministic appears when the Planner supplies `metadata.selectors` — if the demo run never triggers it, show it in the video via a scripted node (acceptable: the assignment requires reporting the layer *chosen*, not all five in one run).
- **Known uncertainty, flagged in-place:** `/v1/cost/by_agent` row shape (Task 5 note), `VisionRequest` field names (Step 1.10) — both have explicit verify-then-adjust instructions.
- **Types:** `_launch_options` used consistently (Tasks 3); `render_session`/`write_report` names match between test and implementation (Task 5).
