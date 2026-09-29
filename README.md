# Signal — a self-marketing agent for content-constrained businesses

**PRD.** [`PRD_v2_Content_Agent.md`](PRD_v2_Content_Agent.md) is the authoritative spec.

## One-line pitch

> A business hands the agent its website URL. The agent reads the site, learns *what they sell, who they sell to, and how they talk*, then sources content, drafts newsletter + Instagram + LinkedIn posts in their voice, and gets sharper every week from their approve/reject feedback.

## What's in the box

- **7 LangGraph nodes**: Crawl → Extract → Seed sources → *(HITL)* Confirm → Persist ▸ Ideate → Evaluate → *(HITL)* Review → Learning → Write → *(HITL)* Publish.
- **Onboarding subgraph** that reads a real website (crawl + trafilatura extract + LLM structured output) and builds a Company Profile with brand voice, content mix, and pillars.
- **Two-branch Ideate** — owned angles (from the profile) + external items (RSS + Tavily), weighted by a `content_mix` value the agent inferred from the site.
- **Deterministic Evaluate** — 5-component score (relevance, novelty, pillar-fit, source-hit-rate, learned-preference-boost), routed feature/uncertain/discard, LLM-only for rationale.
- **Accumulate-then-confirm learning** — structured reject reasons aggregate over a rolling window; stable patterns become *pending rules* the user promotes to *confirmed rules*. Never silently changes anything.
- **Write once, ship many** — one Write call produces newsletter (3 layouts) + IG + LinkedIn in the same voice.
- **Publish** — Buffer (draft mode) for social; Resend for newsletter; `PUBLISH_MODE=preview` mocks everything for demo.
- **Orchestrator chat rail** — supervisor router over a fixed action set (§4.6 of the PRD).
- **Multi-framework eval** — golden-set precision/recall on Evaluate, DeepEval + RAGAS faithfulness on drafts, custom LLM-judge rubric for voice match, approval-rate trend from `feedback_log`, all rolled up into a Markdown report under [`docs/eval-runs/`](docs/eval-runs/).

## Repo layout

```
project-sprint3/
├─ backend/                  Python / FastAPI / LangGraph
│  ├─ app.py                 FastAPI endpoints (§9 of the PRD)
│  ├─ graph/                 LangGraph state + build + nodes
│  ├─ tools/                 crawl · web_search · feeds · buffer · email
│  ├─ memory/                Postgres helpers + schema.sql (Neon)
│  ├─ prompts/               versioned prompt loader
│  ├─ eval/                  golden set · multi-framework eval · report writer
│  ├─ llm.py                 OpenRouter ChatOpenAI factory
│  └─ config.py              env, model list, settings
├─ frontend/                 Next.js 16 App Router + Tailwind
│  └─ src/app/               7 tabs: Review · Sources · Newsletter · Social · Profile · Analytics · Onboard
├─ docs/
│  ├─ architecture.md        graph shape, memory model, node responsibilities
│  ├─ eval-methodology.md    what we measure and why
│  ├─ decisions.md           rationale log for the sharp trade-offs
│  ├─ api-reference.md       pointer to the auto-generated /docs OpenAPI page
│  ├─ prompts/               every LLM prompt used, versioned
│  └─ eval-runs/             one Markdown + JSON file per eval invocation
├─ tests/                    pytest — 104 tests, ~3 min end-to-end
├─ .env / .env.example       API keys + config
└─ pyproject.toml            pytest config
```

## Getting started

### Prerequisites

- Python 3.12
- Node 20+ / npm
- A Neon account (free tier is plenty)
- API keys: **OpenRouter** (LLM), **Tavily** (search), **Buffer** (social), **Resend** (email), **LangSmith** (observability — optional but recommended)

### 1. Configure the environment

Copy `.env.example` to `.env` and fill it in. The DB connection string is already pinned to the Neon project used for development; if you're bringing your own Neon, replace it.

```bash
cp .env.example .env
# then edit .env
```

Key toggles you'll care about:

- `OPENROUTER_MODEL` — the default LLM. Haiku 4.5 is cheap and fast; swap to Sonnet 4.5 if your OpenRouter privacy settings allow it.
- `PUBLISH_MODE=preview` — mocks Buffer + Resend. Flip to `real` when you're ready to actually send.
- `LEARNING_MIN_SAMPLES=3` and `LEARNING_WINDOW=6` — how many reject-reason matches trigger a proposed rule. Higher = more conservative.

### 2. Install and run the backend

```bash
pip install -r backend/requirements.txt
uvicorn backend.app:app --reload
```

The OpenAPI spec lives at [`http://127.0.0.1:8000/docs`](http://127.0.0.1:8000/docs).

### 3. Install and run the frontend

```bash
cd frontend
npm install
npm run dev
```

Open [`http://127.0.0.1:3000`](http://127.0.0.1:3000). First-run: go to the **Onboard** tab and paste your homepage URL.

### 4. Run the tests

```bash
pytest -q
```

The suite talks to a Neon `test` branch (isolated from your main data — see `tests/conftest.py`). 104 tests, ~3 minutes end-to-end.

### 5. Run the eval

```bash
python -m backend.eval.run_eval --slug baseline --company-id demo
```

Writes a Markdown + JSON report under `docs/eval-runs/`.

## Two content mixes, one graph

The PRD calls out a design constraint: some businesses' content is mostly their own story (a wine estate, a local brand), while others' is mostly industry commentary (a startup, a B2B tool). The Ideate node handles both, weighted by a `content_mix` float in [0, 1] the onboarding step infers from the site. This is a **data value, not a fork in the code** — same graph, different weighting.

## The learning loop, spelled out (PRD §6)

Four separations enforced in [`backend/graph/nodes/learning.py`](backend/graph/nodes/learning.py):

1. **Structured feedback, not free-text** — 8 reason codes; free-text note is stored but never drives rules.
2. **Accumulate, don't react** — one reject changes nothing. A stable pattern (≥ `LEARNING_MIN_SAMPLES` of last `LEARNING_WINDOW` rejects sharing one reason) triggers a proposed rule.
3. **Adjust ranking, not sourcing** — learned rules re-weight Evaluate's score; never touch Ideate's breadth.
4. **Confirm the rule, not just the item** — proposed rules land in `pending_rules`; the user confirms via the Profile · Learning tab; only then they hit `confirmed_rules`. The agent *proposes*, the user *disposes*.

Same discipline for sources: hit-rate is a signal, not an axe. Under-performers are flagged; the agent never silently unfollows.

## Eval results at a glance

Headline numbers from the recorded runs in [`docs/eval-runs/`](docs/eval-runs/) — each links to the full report with per-item detail.

**Golden-set routing** (20 hand-labeled items, feature/uncertain/discard):

| Run | Precision | Recall | F1 | TP / FP / FN / TN |
|---|---|---|---|---|
| [Baseline thresholds](docs/eval-runs/demo-baseline-run.md) | **1.00** | 0.25 | 0.40 | 3 / 0 / 9 / 8 |
| [Lowered thresholds](docs/eval-runs/demo-thresholds-lowered-run.md) | 0.71 | **0.42** | **0.53** | 5 / 2 / 7 / 6 |

The trade-off in one line: baseline never features a bad item but under-surfaces; lowering the thresholds nearly doubles recall at the cost of 2 false positives in 20. Production keeps the conservative baseline — a wrongly-featured item costs user trust, an under-surfaced one only costs a scroll to the "uncertain" bucket.

**Onboarding extraction — model comparison** ([full report](docs/eval-runs/demo-models-model-comparison.md), judge: GPT-4o mini, 1–5 per dimension):

| Model | Judge avg | Latency | Cost / run |
|---|---|---|---|
| Claude Haiku 4.5 | 4.8 | 14.5 s | $0.0059 |
| Gemini 2.5 Flash | 4.8 | 5.4 s | $0.0008 |
| GPT-4o mini | 4.8 | 5.4 s | $0.0006 |

Quality ties at 4.8/5 across all three — extraction is prompt-bound, not model-bound — so the pick is a cost/latency call.

**Onboarding extraction — prompt comparison** ([full report](docs/eval-runs/demo-prompts-prompt-comparison.md), same model, v1 vs v2):

| Prompt | Judge avg | Latency | Tokens in/out |
|---|---|---|---|
| v1 — rule-heavy zero-shot | 4.8 | 14.4 s | 2625 / 645 |
| v2 — few-shot with worked examples | 4.8 | 5.5 s | 2694 / 445 |

Same judged quality, but the few-shot variant answers ~2.6× faster with 30% fewer output tokens — worked examples let the model commit instead of deliberating.

## Known limitations

Documented on purpose — these are design gaps, not oversights:

- **Confirmed-rule matching is coarse.** A confirmed rule penalizes a candidate only when the rule's target phrase appears in the candidate's title/angle text ([`evaluate.py`](backend/graph/nodes/evaluate.py) `_pref_boost`). Mapping each reject reason code to a proper scoring signal (e.g. *off-brand* → pillar-fit weighting) is the natural next iteration.
- **`content_history` is keyed by item id alone** ([`schema.sql`](backend/memory/schema.sql)), which is fine for the single-tenant deployment this targets but would collide across companies in a multi-tenant setup.

## Read next

- [`docs/architecture.md`](docs/architecture.md) — the graph, the state, and how HITL interrupts work.
- [`docs/eval-methodology.md`](docs/eval-methodology.md) — what each eval framework measures and why we run multiple.
- [`docs/decisions.md`](docs/decisions.md) — the trade-offs that would be easy to get wrong on a fresh read.
- [`docs/eval-runs/`](docs/eval-runs/) — the eval report history.

## License

All rights reserved.
