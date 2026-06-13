# RUN.md — running the Session 9 browser agent

Windows + PowerShell. You need **two terminals**: one for the gateway (leave it
running), one for the agent.

## Prerequisites

- Python 3.11+ and [`uv`](https://docs.astral.sh/uv/)
- Playwright's Chromium browser binary (installed below)
- Two `.env` files holding provider keys (not committed):
  - `llm_gatewayV9\.env` — needs `ANTHROPIC_API_KEY`, `ANTHROPIC_MODEL=claude-haiku-4-5-20251001`, `GEMINI_API_KEY` (others optional)
  - `S9code\.env` — needs `TAVILY_API_KEY` (for the researcher fallback)
- One-time, first run only:
  ```powershell
  cd llm_gatewayV9 ; uv sync
  cd ..\S9code      ; uv sync ; uv run playwright install chromium
  ```

## Step 1 — Boot the gateway (Terminal 1, leave running)

```powershell
cd "C:\The School Of AI\Session 9 - Browser Agents & Autonomous Web\llm_gatewayV9"
uv run main.py
```
Expect `Uvicorn running on http://0.0.0.0:8109`. The agent calls this for every
LLM/vision request, and `replay_viewer.py` reads its cost ledger.

**Restart the gateway after any change to `agent_routing.yaml`, `router.py`,
`providers.py`, `main.py`, or `pricing.py`** — those load once at startup.

## Step 2 — Smoke-test the gateway (optional, Terminal 2)

```powershell
cd "C:\The School Of AI\Session 9 - Browser Agents & Autonomous Web\llm_gatewayV9"
# Claude chat through the gateway
uv run python -c "from client import LLM; r=LLM().chat(prompt='Reply with exactly: OK', provider='a'); print(r['provider'],'->',r['text'])"
# Claude vision (sends a 200x200 red image)
uv run python -c "import base64,io,httpx; from PIL import Image; b=io.BytesIO(); Image.new('RGB',(200,200),(255,0,0)).save(b,'PNG'); u='data:image/png;base64,'+base64.b64encode(b.getvalue()).decode(); print(httpx.post('http://localhost:8109/v1/vision',json={'image':u,'prompt':'one word colour?'},timeout=90).json().get('text'))"
```
Expect `anthropic -> OK` and `Red`.

## Step 3 — Wipe memory before a clean run (Terminal 2)

```powershell
cd "C:\The School Of AI\Session 9 - Browser Agents & Autonomous Web\S9code"
Set-Content .\state\memory.json "[]" -Encoding utf8
```
**Why:** a populated memory makes the Planner reuse a cached answer (via the
retriever) instead of browsing fresh. Wipe before every demo run. (The FAISS
index files `index.faiss` / `index_ids.json` only exist after an indexing run;
delete them too if present — it is normal for them to be absent.)

## Step 4 — Run the agent with a visible browser (Terminal 2)

```powershell
$env:BROWSER_HEADFUL = "1"        # show the Chromium window
$env:BROWSER_SLOWMO_MS = "400"    # 0.4s between actions, so you can follow
$env:PYTHONUNBUFFERED = "1"       # logs appear immediately

uv run python flow.py "Compare top 3 Hugging Face text-generation models sorted by likes"
```
These env vars live only in the current terminal — set them in the **same**
window you run `flow.py` from, or the browser stays invisible (headless).

What you should see: a 3-node seed plan, a visible Chromium opening
huggingface.co/models and clicking the Text Generation filter + the sort
dropdown, then `distiller → critic → formatter`. If the first browse does not
land the sort, the **critic fails it and recovery re-browses** until the order
is right — that self-correction is expected and is part of the demonstration.
Run ends at the `FINAL:` banner with the comparison table. **Note the session
id** printed at the top.

> Let it run to the `FINAL:` banner. Pressing Ctrl+C mid-run aborts an
> in-flight recovery and prints a `KeyboardInterrupt` traceback — that traceback
> is your interrupt, not a crash.

## Step 5 — Generate the 8-item replay report

```powershell
uv run python replay_viewer.py <session-id>
Start-Process .\state\sessions\<session-id>\replay.html
```
Opens a single self-contained HTML file with all eight trace items. The cost
table is populated only if the gateway (Step 1) is still running.

Walk a run node-by-node in the terminal instead:
```powershell
uv run python replay.py <session-id>     # Enter=next, p=prompt, o=output, q=quit
```

## Step 6 (optional, for the video) — show the "blocked" layer

Point the agent at a login/CAPTCHA-walled page to see the cascade return
`gateway_blocked` and recovery re-plan around it:
```powershell
uv run python flow.py "From https://www.instagram.com/accounts/login extract the trending posts"
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| Gateway startup doesn't list `anthropic` | `ANTHROPIC_API_KEY` missing from `llm_gatewayV9\.env` |
| No browser window, run still proceeds | `BROWSER_HEADFUL` not set in the **same** terminal as `flow.py` |
| `[memory.read] N hit(s)` and Planner skips browsing | re-do Step 3 (wipe memory) |
| `502/503` bursts on a skill | provider rate-limited; check `agent_routing.yaml` pin and the gateway's failover ladder |
| `net::ERR_CONNECTION_RESET` on a site | transient; the browser retries `goto` 3× with backoff — re-run if it persists |
| `WinError 5: Access is denied` during save | Defender scanning state files mid-rename → admin PowerShell: `Add-MpPreference -ExclusionPath "<repo path>"` |
| `memory.remember … 502` warning | harmless (Gemini free-tier classifier quota); memory falls back to a deterministic write |
| `I/O operation on closed pipe` at exit | cosmetic Windows MCP-shutdown artifact; ignore |
