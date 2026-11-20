# Lowball — a negotiation agent that remembers every customer

A B2B negotiation AI for manufacturers. A manufacturer has thousands of customers who
each negotiate differently — some grind hard, some agree fast, some stall. The agent
negotiates with each one, **remembers every customer**, and adapts its approach from
that memory. Built at HackNation (tracks: **haggle · compare · call**).

The differentiator is **memory**: a Kuzu graph + native vector index in one engine.
Everything connects — customers, deals, calls, tactics — and that graph makes the agent
smarter per customer. This is **retrieval-augmented adaptation, not model training**: the
agent adapts per customer from memory; the model itself is never retrained.

## Architecture

```
memory/        Kuzu graph + vector index — the foundation. get_context() / write_call().
negotiation/   The agent, the customer bot, scoring, the arena loop, terminal demo.
frontend/      Flask dashboard + messaging views (own localhost :5001).
tests/         End-to-end smoke test.
```

```
Customer ──HAS_DEAL──▶ Deal ◀──ABOUT── Call ◀──HAD_CALL── Customer
   │                                                          
   ├─EXHIBITS─▶ Pattern ─WORKED─▶ Customer      (tactics that worked)
   └─SIMILAR_TO─▶ Customer                       (lookalikes → tactic transfer)
```

Before a negotiation the agent calls `get_context(customer)` → style, floor price,
past-call summaries, winning tactics (incl. tactics transferred from lookalike
customers), and an escalation flag. After, `write_call()` stores a short summary +
outcome score. A run of declining scores → the agent hands off to a human.

## Setup

```bash
python3.12 -m venv .venv           # kuzu has no wheel for 3.14 yet — use 3.12
.venv/bin/pip install -r requirements.txt
.venv/bin/python -m memory.seed    # build the graph with demo data
```

Optional — go live (otherwise an offline mock runs, deterministic):
```bash
cp .env.example .env               # add OPENAI_API_KEY (model: gpt-5.6-terra)
```

## Run

**Terminal demo** — watch the agent negotiate all 3 styles, reasoning printed each turn:
```bash
.venv/bin/python -m negotiation.demo
```

**Web dashboard** — manufacturer's screen (two views, one server on :5001):
```bash
.venv/bin/python -m frontend.app
#   http://127.0.0.1:5001/            dashboard (overview, metrics, memory drawer)
#   http://127.0.0.1:5001/messaging   live turn-by-turn negotiations, 3 side by side
#   http://127.0.0.1:5001/graph       3D memory graph — every node connected, click for detail
```

**Smoke test** — verify the whole stack (no API key needed):
```bash
.venv/bin/python -m tests.smoke
```

## The demo in one minute

Same product for everyone — **Cotton T-shirts, floor 3.20 / target 4.00 USD**. Only the
customer changes, so the outcome differences come purely from adaptation:

| customer | style | what the agent does | outcome |
|----------|-------|--------------------|---------|
| Alpha Textiles | hard haggler | anchors high, grinds, holds above floor 3.20 | closes 3.68 |
| Bravo Imports | responsive | rapport, closes fast near target | closes 4.16 |
| Charlie Retail | staller | sees declining history → **escalates to human** | handed off |
| Delta Fashion | lookalike | wins using a tactic **transferred from Alpha** | closes 3.68 |

Guardrails are enforced in code, not left to the model: never quote below floor,
always propose a next step, escalate on declining outcomes.

## Known limitations
- `memory/embeddings.py` is a deterministic stub, so vector search isn't semantic yet. Swapping in `all-MiniLM-L6-v2` needs no schema change (`EMBED_DIM=384` already matches).
- Without an `OPENAI_API_KEY`, an offline mock generates the reasoning. It proves the logic but isn't model-generated.

## Status
- Working: memory layer, negotiation arena, guardrails, escalation, scoring, both web views
- Tested: 25-check smoke test passing on the offline mock
- Not yet verified: live OpenAI calls (needs a key) and the ElevenLabs voice call (track 3)
