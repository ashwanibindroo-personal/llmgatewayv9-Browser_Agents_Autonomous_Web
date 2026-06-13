# YouTube Demo Script — Session 9 Browser Agent

Target length ~6–8 minutes. Two terminals + a browser window + the
`replay.html`. Record at 1366×900 so the headful Chromium and the terminals are
both legible.

## Pre-roll checklist (do NOT record)
- Gateway running in Terminal 1 (`uv run main.py`, shows `:8109`).
- Memory wiped: `Set-Content .\state\memory.json "[]" -Encoding utf8`.
- Env vars set in Terminal 2: `BROWSER_HEADFUL=1`, `BROWSER_SLOWMO_MS=400`, `PYTHONUNBUFFERED=1`.
- `replay.html` from the golden trace open in a tab, ready to show if the live run wanders.

---

## Scene 1 — The idea (45s, talking head / slides)
> "This is Session 9 of EAG V3. The agent is a *growing graph* — a Planner draws
> a to-do board of skill nodes, and an Executor runs every node whose inputs are
> ready, in parallel. New this session: the agent can drive a real web browser.
> The rule I want you to hold onto — **the graph is the loop**. There's no
> while-loop in the code; the behavior emerges from the plan."

Show the three-box diagram from the README (agent ↔ gateway, agent → browser).
> "Three programs: the agent, a gateway that's a switchboard to several AI
> vendors, and a browser package that drives Chromium. And one hard rule: I never
> edit the orchestrator. New ability is just data — a config block and a prompt."

## Scene 2 — The cascade (45s, slide)
> "The browser uses a cost ladder. Reading HTML is free; asking a vision model to
> look at screenshots is expensive. So it tries the cheapest sense first —
> extract, then deterministic clicks, then the accessibility tree, then vision —
> and a CAPTCHA or login wall is a first-class answer called *blocked*, not a
> crash."

Point at the five badges (extract / deterministic / a11y / vision / blocked).

## Scene 3 — Launch (30s, Terminal 2)
Type and run:
```powershell
uv run python flow.py "Compare top 3 Hugging Face text-generation models sorted by likes"
```
> "I ask it to compare the top 3 Hugging Face text-generation models by likes.
> Watch the console: the Planner emits a three-node seed plan — browse the list,
> distill the fields, format the table."

## Scene 4 — The visible browser (90s, Chromium window)
> "Here's the actual browser. It's not scraping a static snippet — it's
> *interacting*."

Narrate the visible actions as they happen (slow-mo makes this easy):
> "It clicks the **Text Generation** task filter… opens the **Sort** dropdown…
> chooses **Most likes**. That's three visible interactions on a live, dynamic
> page — exactly what the assignment asks for."

## Scene 5 — The graph grows / self-corrects (60s, Terminal 2)
As critic + recovery lines scroll:
> "Now the interesting part. A Critic node is auto-inserted after the distiller.
> If the browse didn't actually apply the *likes* sort, the Critic catches that
> the order is wrong and *fails* it — which makes the graph grow a recovery
> Planner that re-browses. You're watching the agent **correct itself**, live.
> This is the growing graph: the plan changes at runtime in response to what
> happens."

Wait for the `FINAL:` banner.
> "And there's the final comparison table — DeepSeek-R1, Llama-3-8B,
> Llama-3.1-8B-Instruct, sorted by likes."

## Scene 6 — The replay report, all 8 items (120s, browser)
```powershell
uv run python replay_viewer.py <session-id>
Start-Process .\state\sessions\<session-id>\replay.html
```
Scroll through and call out each of the eight tracked items:
> "One self-contained HTML file. **(1)** the original goal. **(2)** the Planner
> DAG, with a badge showing which cascade layer each browser node used. **(3)**
> the cascade path — here, a11y. **(4)** the exact actions, turn by turn.
> **(5)** evidence — the actual screenshots the agent took, embedded right in
> the page. **(6)** the raw extracted text. **(7)** the final comparison table.
> **(8)** turn count and a token-and-cost summary per agent, straight from the
> gateway's ledger."

## Scene 7 — Optional: the blocked layer (45s)
```powershell
uv run python flow.py "Get the top 5 posts from the r/LocalLLaMA subreddit at https://www.reddit.com/r/LocalLLaMA/top/ and list their titles"
```
> "To show the *blocked* layer deliberately: this reads like a normal content
> request, but Reddit serves an automated browser a reCAPTCHA. Watch — the
> browser node fails in about 12 seconds with `gateway_blocked (recaptcha)`.
> The cascade recognized the wall instead of getting stuck on it, and the agent
> hands back gracefully. Failure is handled as data."

A committed example of this run is in `traces/s8-960b2d46/replay.html` — its
DAG shows the browser node with a red **blocked** badge.

## Scene 8 — Wrap (45s, talking head)
> "Everything you saw plugged into a frozen orchestrator. `flow.py` is
> byte-identical to last session. No agent framework — just Playwright and our
> own logic. The whole capability is a config block, some prompts, a provider
> port, and a report generator. Code and the full trace are in the repo. Thanks
> for watching."

---

### If the live run wanders during recording
The a11y driver occasionally needs a couple of recovery cycles to land the sort
(that's Scene 5's story, so it's fine to show). If it genuinely stalls, cut to
the committed golden trace: open `traces/s8-5d301d39/replay.html` and narrate
Scene 6 from it — it's a complete, correct run.
