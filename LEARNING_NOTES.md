# Learning Notes — Session 9, in plain English

A teaching companion to the code. Concepts first, then the debugging story —
because the bugs taught more than the features.

## 1. The graph IS the loop

A classic agent is a hand-written `while` loop: think → act → think → act,
one step at a time, in single file. Session 8 replaced that with a **to-do
board with arrows**. The Planner doesn't *do* anything — it pins sticky notes
(skill nodes) and draws arrows (dependencies). The Executor runs the only rule
left in the system:

> *"Which notes have all their incoming arrows satisfied? Run all of them now,
> in parallel. Repeat until the board is empty."*

Everything else falls out for free:
- **Parallelism** without parallel code — independent notes run together.
- **"Looping" without a loop** — the system keeps going because notes remain on
  the board, not because code says `while`. Different question → different board
  → different behavior. The intelligence moved out of the code and into the plan.
- **Runtime growth** — finished notes can pin successors; a `critic:true` skill
  auto-spawns a Critic on its edges; a *failed* note pins a recovery Planner
  that draws a detour. Nobody rewrites the loop; they add paper to the board.
- **Crash-proofness** — the whole agent state *is* the board, photographed to
  disk after each step. Kill it, reload the photo, finish the unfinished notes.

Proof it works: `flow.py` is byte-identical between the Session 8 base and this
Session 9 browser agent. The agent learned to drive a browser; the engine never
knew.

## 2. A skill is data, not code

A capability = one YAML block + one prompt `.md`. No Python class. That's why
"add browser automation without touching the orchestrator" is even possible —
and why every fix in the debugging story below was a prompt or config edit, not
an engine change.

## 3. The browser cascade is a cost ladder

Reading a page's HTML is nearly free. Asking a vision model to stare at
screenshots costs real money and time. So the Browser skill tries the cheapest
sense first and climbs only when forced: **extract → deterministic → a11y →
vision**. A goal that says "filter" or "sort" can't be satisfied by reading
static HTML, so it forces a climb to the interactive layers. And a CAPTCHA or
login wall is treated as a real answer — **blocked** — not a crash, so the
agent can route around it.

Two senses worth distinguishing:
- **a11y** = read the page the way a screen reader does — a text list of
  clickable things. Cheap (no images), and enough for most navigation.
- **vision** = annotate a screenshot with numbered boxes and ask the model
  "which number?". Expensive, the last resort for pages text can't describe.

## 4. The debugging story (the real lessons)

Getting the demo to produce a *correct* answer took six runs. Every failure was
fixed with **data, not engine code** — and the browser's own saved screenshots
were the debugging instrument each time.

| Run | What broke | Root cause | Fix (all data/config) |
|---|---|---|---|
| 1 | 502/503 storms + connection resets | 4 parallel browser nodes hard-pinned to Gemini blew its 15 rpm free tier; a *pinned* call can't fall back | route browser → Anthropic (100 rpm, vision-capable) via `agent_routing.yaml` |
| 1 | URL-less browser nodes | Planner fanned out per-model browsers with *guessed* URLs before anything read the list | two-wave plan: browse list → distiller emits per-model nodes once real URLs exist |
| 2 | `ERR_CONNECTION_RESET` | old headless Chromium's network fingerprint contradicts the spoofed Chrome user-agent; HF resets it | new-headless `channel="chromium"` + `goto` retries with backoff |
| 3 | browser stuck 157s, step cap | first a11y snapshot fired before the JS list hydrated — driver saw only the nav and wandered | wait for `networkidle` before the first snapshot |
| 4 | wandered 12 turns with the filter visible | the small model is poor at *inventing* multi-step UI strategy from a vague goal | write goals as a **procedure** ("click X, then open Y, then choose Z"); raise step budget 12→18 |
| 5 | driver quit at the finish line | it thought *it* had to read the page content; it didn't know the harness harvests page text after `done` | tell both driver prompts: reaching the goal *state* IS success for extract goals |
| 6 | correct table, but WRONG final answer | a gated Llama detail page wandered → recovery storm → a degraded researcher-fallback formatter ran **last**, and the engine keeps the *last* formatter's output | collapse to a **single-wave, single-formatter** plan; the listing page already has every field but license |

**Cross-cutting lessons:**
- **Reproduce before fixing.** A 4-variant probe matrix turned "it's flaky"
  into "headless + spoofed-UA gets reset; headful doesn't" — and revealed the
  fix wasn't a *better* disguise but *tolerance* (retries).
- **Verify the test, not just the system.** A vision smoke test returned
  "White" for a 1×1 red pixel — the pixel was too small to perceive, not the
  pipeline broken. A 200×200 image returned "Red" instantly.
- **The frozen engine has quirks you must design around, not fix.** "Last
  formatter wins" can't be changed (`flow.py` is off-limits), so the fix was to
  guarantee exactly one formatter ever runs.
- **A strict judge is a feature.** In the golden run the Critic rejected the
  distiller three times because the browse hadn't actually applied the *likes*
  sort — and recovery re-browsed until it had. The S8 maxim "LLMs can't count"
  applies to ordering too; the Critic enforced it.

## 5. Why route cognitive skills to Claude

The gateway lets each skill pick a provider by name. We pin the reasoning-heavy
skills (planner, distiller, formatter, verifier, and the browser driver loops)
to Claude Haiku 4.5 — vision-capable, reliable tool-use, and generous rate
limits that survive the parallel bursts a growing graph produces. The
config-driven routing is itself a lesson: when Gemini's free tier throttled the
browser, the fix was *one line* in a YAML file, not a code change.

## 6. The payoff

The replay viewer folds the persisted run + screenshots + cost ledger into one
HTML file showing all eight required items. That same persistence + evidence
machinery — built for the deliverable — is what made the agent debuggable in
the first place. Observability wasn't an afterthought bolted on for grading; it
was the tool that got the demo working.
