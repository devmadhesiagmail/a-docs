Engineering onboarding · Claims pilot

# How the *UnderCover* claims platform works

A walkthrough of the codebase: what the business needs, how the three parts fit together, how one claim moves through the system, and why it was built this way. Read this before your first change.

Branch: week1-day3-dev Python 3.12 · uv Node 26 FastAPI · SQLite · LangChain · React

Contents

1. [The business problem](#business)
2. [Architecture](#architecture)
3. [The claim lifecycle](#lifecycle)
4. [One claim, end to end](#flow)
5. [Design decisions](#decisions)
6. [Code tour](#code)
7. [The API](#api)
8. [Testing and quality gates](#quality)
9. [How we work](#process)
10. [Your first hour](#run)
11. [Things to watch](#watch)
12. [Where it goes next](#roadmap)

## 01The business problem

UnderCover Travel Insurance is a fictional travel insurer, and our customer. Read `docs/SPEC.md` for the full brief from their head of claims operations.

Three claims handlers work a queue of claims by hand. For each one they read the customer's story and documents, check them against the policy and the policy wording, and then **approve** with a payout, **decline**, or **refer** it for a person's judgment.

The pilot adds the **UnderCover Agent**, an LLM agent that works the queue the way a handler does and explains each decision in two or three sentences, citing the relevant policy section. Handlers watch it on a dashboard and sign off large payouts.

Hard rule

#### Over $1,000 needs sign-off

The agent may approve, but the system must block the payment until a handler signs off, even if the agent tries to pay.

Hard rule

#### The agent never sees the answers

`data/evals/` holds the handlers' real decisions. It's our test set, so nothing in the product opens it.

Scope

#### Deliberately small

No sign-in, no real payments, no customer messages, and no claims manual yet. The agent decides from the policy wording alone.

## 02Architecture

Three parts with one rule: **the backend is the only thing that touches data**. The agent and the dashboard are both API clients.

Neither the agent nor the dashboard reads a file or the database directly. Even the agent's instructions come from the backend, at `GET /api/instructions`.

## 03The claim lifecycle

Every claim is in one of four states. Transitions happen only through backend operations, each inside a `BEGIN IMMEDIATE` transaction, so two writers can't race on one claim.

### Payment is separate from deciding

`pay_claim(amount)` doesn't change the status. It records a payment only when *all* of these hold, and otherwise returns **409** with a plain-English reason. See `payment_refusal()` in `backend/main.py`.

#### The decision is approve

Declined, referred or undecided claims have nothing to pay.

#### The amount equals the payout

The agent can't pay a different number than it approved.

#### Not already paid

Paying twice is refused.

#### Within the limit, or signed off

The limit is `payout_approval_limit` in `data/company.json`, never hard-coded.

## 04One claim, end to end

What happens when you run `uv run -m agent CLM-1002`, a claim whose approval comes to $1,150.

1. CLI · agent/\_\_main\_\_.pyPicks the claim ids (the ones you named, or every *waiting* claim in queue order), makes a fresh `thread_id` (a UUID) for each, and calls `run(claim_id, thread_id)`.
2. agent/main.py · run()Fetches `GET /api/instructions`: `backend/instructions.md` with `data/policy_wording.md` appended. That text becomes the system prompt.
3. Backend · start_claim`POST /start` moves the claim from *waiting* to *in progress* and records a "start" step. If the claim is already decided, the backend refuses and the CLI prints why.
4. Agent loop · LangChain create_agentThe model calls `get_claim`, then `get_policy` and `get_claim_history`, using the ids it found in the claim. Each tool is a thin HTTP call to the backend.
5. Step logging · run()As the graph streams updates, `run()` posts every tool call, tool result and message to `POST /steps`. The dashboard, polling every 2 seconds, shows them live.
6. Model · decide_claimWorks out the payout from section 10 of the wording (per-item caps, minus what others paid, minus the deductible, capped at the maximum) and records approve for $1,150 with a reason and a policy section. The backend sees it's over $1,000 and sets *waiting for sign-off*.
7. Model · pay_claimThe instructions tell it to pay after every approval. The backend refuses with 409, and the tool returns `Refused: A payout over $1,000.00 needs a handler's sign-off…` as text, so the run ends cleanly and the refusal is visible in the steps (in Raspberry).
8. Handler · dashboard SignOff.tsxChooses a handler who may approve over the limit and clicks **Sign off and pay $1,150.00**. That calls `sign_off_claim` (→ *decided*), then `pay_claim`, which now succeeds.

## 05Design decisions, and why

The ten decisions in `docs/PLAN.md` are the most useful thing to understand. Here they are grouped by what they protect.

#### Business rules live in the backend, not the prompt

The spec says to block the payment "even if the agent tries to pay". A prompt is a request, not a guarantee. Enforcing the rule in `pay_claim` means no model mistake, prompt injection or UI bug can get round it.

#### Agent reaches the world only through the API

Five tools, each one wrapping one operation, and none reads a file. That's what keeps `data/evals/` out of reach and makes the agent's capabilities easy to audit.

#### Instructions are served by the backend

The agent and the dashboard read the same text from `/api/instructions`, so handlers see exactly what the agent sees. The wording goes inline rather than behind a tool because it's short and every claim needs it.

#### Refusals come back to the model as text

LangChain's tool node re-raises errors, which would end the run. Returning `Refused: <reason>` lets the agent read why and finish the claim.

#### run() logs the steps, not the model

There's no "log this" tool. The harness posts every message it streams, so the model can't skip or misreport a step. That's our audit trail.

#### Least personal data

`get_claim_history` returns name, customer since and earlier claims only. Email, phone, date of birth and address are never served.

#### SQLite with plain sqlite3, records stored as JSON

A handful of tables, no ORM. Each record from `data/` is stored whole, with workflow columns (status, decision, sign_off, payment) beside it. This is simple now, and it moves to Supabase in Week 2.

#### Polling, not websockets

A refetch every 2 seconds is live enough to watch an agent work, with far less machinery.

#### Hash routing, no router library

Three pages: `#/`, `#/claims/CLM-1001`, `#/instructions`. Links and the back button work.

#### End-to-end tests get their own world

Playwright runs its own backend on :8001 with a fresh DB in the temp folder, and its own dashboard on :5174, so `npm run check` never wipes decisions that cost API credit.

## 06Code tour

The product is about 600 lines of Python and 1,000 lines of TypeScript, plus tests. You can read all of it in an afternoon.

### Backend

backend/main.py

All 14 operations, the Pydantic bodies (`Decision` validates that the payout fits the decision), `payment_refusal()` and `refuse()` for 409s. Start here.

backend/db.py

The schema as one `RESET` script, `connect()` (path from `UNDERCOVER_DB`) and the `transaction()` helper.

backend/seed.py

Loads six named JSON files from `data/` (never evals) and starts every claim as waiting. The app also seeds itself on startup if the DB file is missing.

backend/instructions.md

The agent's system prompt: the seven steps for working a claim. Changing agent behaviour usually means editing this, not code.

### Agent

agent/main.py

The five `@tool`s, `model()` (OpenRouter through `ChatOpenAI`), `steps()` to turn messages into steps, and `run()`. `run()` takes an optional `llm`, which is how tests inject a stub.

agent/\_\_main\_\_.py

The CLI: `waiting_claims()`, `work()`, and a one-line `outcome()` per claim.

### Dashboard

dashboard/src/api.ts

Types that mirror the API, `getJson`/`postJson` (which turn a 4xx `detail` into an Error), and the `useJson(path, pollMs)` polling hook every page uses.

App.tsx · Queue.tsx · ClaimPage.tsx

The hash router and the three pages. `ClaimPage` composes the header, the decision panel, the story, the policy, the history and `Steps`.

Steps.tsx · SignOff.tsx

Live steps (results folded, refusals in Raspberry) and the sign-off form (sign-off, then pay).

format.ts · styles.css

Money and dates, and the brand from `docs/BRAND.md` in light and dark mode.

### Everything else

docs/

The source of truth: SPEC (the customer's needs), TECH_STACK, CODE_STANDARDS, PROCESS, BRAND, PLAN and STATUS. Read all of it before changing anything.

data/ · context/

UnderCover's records. `context/` holds data governance's files, which become the context layer in Week 3.

e2e/

Playwright specs: queue, instructions, theme and claim. The claim specs run after the read-only ones because they change state.

labs/ · guides/ · setup/

Course material and briefs for later builds. These aren't part of the running product.

## 07The API

Every operation has an `operation_id`, a one-line docstring and no optional inputs, which keeps it ready to expose as agent tools or an MCP server later. Errors: 404 for unknown ids, 409 when the state doesn't allow a step (with a reason), and 422 for a bad body.

| Method | Path | operation_id | Used by | Does |
| --- | --- | --- | --- | --- |
| GET | `/api/claims` | `list_claims` | Dashboard, CLI | The queue in order, with status and decision |
| GET | `/api/claims/{id}` | `get_claim` | Agent, dashboard | The claim in full, plus decision, sign_off and payment |
| GET | `/api/policies/{id}` | `get_policy` | Agent, dashboard | The policy with its plan limits and add-ons |
| GET | `/api/customers/{id}/history` | `get_claim_history` | Agent, dashboard | Name, customer since, earlier claims (no contact details) |
| GET | `/api/handlers` | `list_handlers` | Dashboard | The handlers who can sign off |
| GET | `/api/instructions` | `get_instructions` | Agent, dashboard | The instructions followed by the policy wording |
| POST | `/api/claims/{id}/start` | `start_claim` | run() | Waiting or in progress → in progress |
| POST | `/api/claims/{id}/steps` | `add_step` | run() | Records a call, result or message |
| GET | `/api/claims/{id}/steps` | `list_steps` | Dashboard | The steps, oldest first |
| POST | `/api/claims/{id}/decision` | `decide_claim` | Agent tool | Records the decision once; over the limit → waiting for sign-off |
| POST | `/api/claims/{id}/payment` | `pay_claim` | Agent tool, dashboard | Pays if every rule holds, otherwise 409 |
| POST | `/api/claims/{id}/sign-off` | `sign_off_claim` | Dashboard | Handler sign-off → decided (403 if the handler isn't allowed) |

With the backend running, the interactive docs are at `http://localhost:8000/docs`.

## 08Testing and quality gates

The bar is high on purpose. A coding agent writes much of this code, and the gates are what keep it honest.

Python

#### ruff · pyright · pytest

Lint and format with a strict rule set (including security and complexity rules), types checked, and at least 80% coverage. Each test gets its own temporary database.

TypeScript

#### ESLint · Prettier · tsc · vitest

Strict mode and never `any`. Components are tested against a stubbed API, with at least 80% coverage.

End to end

#### Playwright

Real browser, real backend, isolated ports and DB. Claims are set up through the API, never with a model.

### Rules that aren't negotiable

- **Tests never call a model.** Agent tests drive a scripted stub through `run(..., llm=stub)`, so every path is checked for free and repeatably.
- The **lefthook pre-commit hook** formats staged files and runs the checks for the part you changed. Never use `git commit --no-verify`.
- **Never lower a threshold**, turn off a rule or exclude a file to get green. If a rule looks wrong, raise it.
- Run `npm run check` before handing work over, and keep `docs/STATUS.md` current.

## 09How we work

We work as forward deployed engineers. The documents in `docs/` are the customer's requirements, and we deliver in reviewed phases.

1. BriefA brief arrives in `docs/` (the pilot used `SPEC.md`; later builds use files such as `HARNESS.md` and `DURABLE.md`). Build only what `docs/` asks for, never ahead.
2. Plan"Please plan …" writes `docs/PLAN_<NAME>.md` with no code: 5 or 6 phases, the decisions and why, and checkbox success criteria you can verify yourself.
3. ReviewA person reads and edits the plan. This is the cheapest point to change direction.
4. ImplementEach phase: build with tests, check every criterion, tick the boxes, update STATUS, and commit (the hook runs the checks).
5. Hand overA full `npm run check`, and the results reported as they are.

### Code standards, in full

> Keep it simple. Comments only when necessary. Avoid overly defensive code. Always uv run, never python3. Strict TypeScript, never any.docs/CODE_STANDARDS.md

You can see it in the code: no try/except in the backend except where a 4xx has to become text, no ORM, no state library, and docstrings of one line.

## 10Your first hour

From a fresh clone to watching the agent decide a claim.

```
# 1. Install (Python 3.12 via uv, Node 26)
uv sync
npm install            # also installs the git hook and Playwright's Chromium

# 2. Add your key to .env
OPENROUTER_API_KEY=sk-or-...
MODEL_NAME=openrouter/free   # optional: the free model, no spend

# 3. Run the backend (:8000) and the dashboard (:5173)
npm run dev

# 4. In another terminal, let the agent work one claim, then watch it on the dashboard
uv run -m agent CLM-1001
uv run -m agent         # every waiting claim, in queue order (costs API credit)

# 5. Reset the queue from data/, and run every check
npm run seed
npm run check
```

A good first exercise: open `http://localhost:5173/#/claims/CLM-1002`, run the agent on it, and follow steps 1 to 8 above as they happen. Then read `backend/tests/test_workflow.py`, which states every rule as a test.

## 11Things to watch

Known limits of the pilot. Most are deliberate for now. Know about them before you build on top.

Ops

**A crashed run leaves a claim "in progress"**`uv run -m agent` with no arguments only picks up *waiting* claims. To recover a claim, name it explicitly: `start_claim` allows a restart.

Config

**The agent's backend URL is hard-coded**`BACKEND_URL = "http://localhost:8000"` in `agent/main.py`. The dashboard reads `BACKEND_URL` from the environment, but the agent doesn't yet. This matters when we deploy.

Money

**Amounts are floats**Payout and payment are compared exactly (`amount != payout`). That works because the agent sends back the same number, but integer cents or Decimal would be safer before real payments.

UX

**Sign-off and pay are two calls**If the pay call fails after the sign-off succeeds, the claim is decided but unpaid. The form handles this by showing a plain *Pay* button.

Security

**No authentication, and the story is untrusted input**Anyone who can reach the API can sign off. A customer's story goes straight into the model's context, and `data/security/` holds a prompt-injection claim for exactly this. The backend payment rule is our main defence today; auth arrives in Week 2 Day 5 and an injection check in Week 3 Day 2.

Accuracy

**Nothing scores the agent yet**The live runs prove the wiring works, not that decisions are right. Evals against `data/evals/` arrive with the swarm build. Until then, treat agent accuracy as unmeasured.

## 12Where it goes next

Each build day adds a layer on top of this one. Ed's reference build for each stage is on `week<N>day<N>-end` branches.

W1 D3 · nowThe pilot: backend, dashboard and a first agent

W1 D5The agent's harness: tools, middleware, instructions and a searchable claims manual

W2 D3Durable state: Supabase, and the claim agent as a LangGraph graph that survives restarts

W2 D4A dispatcher with a swarm of up to 25 claim agents, and evaluations

W2 D5Versioned agent configuration, an improver agent and handler sign-in

W3 D2Agent API key, hidden-instruction check, and the app ready for AWS Lambda

W3 D3The context layer as an MCP server, agent memory and AgentCore Gateway

W3 D4The agent on AgentCore Runtime, with Sign off or Send back

W3 D5The UnderCover Copilot: a chat for handlers in the dashboard

The pilot's choices, such as API-first tools, operation_ids, rules in the backend and steps logged by the harness, are what make these layers cheap to add.