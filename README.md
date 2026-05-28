# Fridge Reconciler — Live Demo Repo

A toy [DriftlessAF](https://github.com/driftlessaf/go-driftlessaf) reconciler built
for the Chainguard × Elastic meetup:
**"An open source agentic reconciliation framework proven at scale."**

Same path-reconciler pattern Chainguard uses in production to keep 2,000+ container
images in sync with upstream — applied here to dinner.

---

## What this repo is

- **`fridge.yaml`** — observed state. What's in your fridge right now.
- **`goal.yaml`** — desired state. The meal vibe + constraints (vegetarian, ≤25 min, etc).
- **`.github/chainguard/dev-mgreau-fridge-reconciler.sts.yaml`** — OctoSTS trust policy
  granting the Cloud Run reconciler permission to write Check Runs.

Editing `fridge.yaml` (or `goal.yaml`) and pushing to `main` triggers a deployed
fleet of four AI agents that reconcile the gap between desired and observed state
and post the resulting recipe as a GitHub Check Run on the commit.

---

## Architecture

```
                                                  GCP
                            ┌──────────────────────────────────────────────────────────────────┐
                            │                                                                  │
 ┌────────────┐             │  ┌──────────────┐      ┌────────────────────┐                    │
 │  GitHub    │   push      │  │ github-events│      │ CloudEvents Broker │                    │
 │            │             │  │  (webhook)   │      │ (filter + enqueue) │                    │
 │ fridge.yaml├─────────────┼─►│              ├─────►│                    │                    │
 │ goal.yaml  │             │  └──────────────┘      └─────────┬──────────┘                    │
 │            │             │                                  │                               │
 │            │             │                                  ▼                               │
 │            │             │                         ┌────────────────────┐                   │
 │            │             │                         │ Workqueue          │                   │
 │            │             │                         │ (rcv + dispatcher) │                   │
 │            │             │                         └─────────┬──────────┘                   │
 │            │             │                                   │                              │
 │            │             │                                   ▼                              │
 │            │             │  ┌──────────────┐     ┌────────────────────┐   ┌───────────────┐ │
 │            │             │  │ OctoSTS      │◄────┤ Reconciler         ├──►│  Metaagent    │ │
 │            │             │  │ (get token)  │     │ (convergence loop) │   │  (Vertex AI)  │ │
 │            │             │  └──────────────┘     └────────────────────┘   │               │ │
 │            │             │                                                │  Inventory    │ │
 │            │             │                                                │     │         │ │
 │            │             │                                                │     ▼         │ │
 │            │             │                                                │  Chef ⇄ Critic│ │
 │            │             │                                                │     │         │ │
 │            │◄────────────┼─── Check Run with recipe via GitHub API        │     ▼         │ │
 │            │             │                                                │   Judge       │ │
 │ Check Run +│             │                                                └───────────────┘ │
 │ Recipe md  │             │                                                                  │
 └────────────┘             └──────────────────────────────────────────────────────────────────┘
```

**Flow:**

1. GitHub sends webhook when `fridge.yaml` is pushed
2. `github-events` converts the webhook into a CloudEvent
3. CloudEvents Broker filters by path pattern and routes to Workqueue
4. Workqueue dedups by file path; dispatcher picks up the work item
5. Reconciler mints a GitHub token via OctoSTS, then fetches `fridge.yaml` + `goal.yaml`
6. Reconciler runs the **convergence loop** through the Metaagent on Vertex AI:
   - **Inventory** parses the raw YAML into a normalized state (expiry buckets, days-to-expiry)
   - **Chef** proposes a recipe given the goal + inventory
   - **Critic** emits structured drift signals (missing / insufficient / expired ingredients, constraint violations)
   - Chef ⇄ Critic iterate (up to 3 times) until `drift.count == 0`
   - **Judge** scores the converged recipe on a 0.0–1.0 quality scale
7. Reconciler writes a GitHub Check Run on the commit, with the recipe rendered as Markdown in the summary

Same plumbing as Chainguard's `update-bot` (workqueue + OctoSTS + reconciler) — only the output sink differs (Check Run instead of PR).

## Observability — every agent step in Elastic APM

```
   ┌─── per agent execution ─────────────────────────────────┐
   │                                                          │
   │   metaagent ─► Vertex AI (Claude)                        │
   │       │                                                  │
   │       ├─ emits OTel spans:                               │
   │       │     • invoke_agent      (root per agent call)    │
   │       │     • execute_tool submit_result                 │
   │       │       attrs: gen_ai.input.messages = JSON,       │
   │       │              driftlessaf.tool.reasoning = text,  │
   │       │              gen_ai.output.messages = result     │
   │       │                                                  │
   │       └─ emits GenAI metrics:                            │
   │             • gen_ai.client.token.usage                  │
   │             • genai.tool.calls                           │
   │                                                          │
   └──────────────────────┬───────────────────────────────────┘
                          │
                          ▼  OTLP/HTTP (Authorization: ApiKey ...)
   ┌─────────────────────────────────────────────────────────┐
   │             Elastic Cloud Serverless APM                 │
   │                                                          │
   │   Trace tree per reconcile:                              │
   │       reconcile (gen=hash)                               │
   │        ├── inventory.parse                               │
   │        ├── iter-1                                        │
   │        │    ├── chef.propose                             │
   │        │    │    └── invoke_agent                        │
   │        │    │         └── execute_tool submit_result     │
   │        │    └── critic.evaluate                          │
   │        │         └── invoke_agent                        │
   │        │              └── execute_tool submit_result     │
   │        ├── iter-N  …                                     │
   │        └── judge.score                                   │
   │             └── invoke_agent                             │
   │                  └── execute_tool submit_result          │
   │                                                          │
   │   Root span attributes (per reconcile):                  │
   │     fridge.generation                  hash             │
   │     fridge.status                      CONVERGED / …     │
   │     fridge.iterations_to_converge      1, 2, 3           │
   │     fridge.judge_score                 0.10 – 1.00       │
   │     fridge.recipe_title                "Spinach…"        │
   │     fridge.recipe_prep_minutes         20                │
   │     fridge.recipe_ingredients          count             │
   │                                                          │
   │   Dashboard ➜ panel queries in queries.md (private)      │
   └─────────────────────────────────────────────────────────┘
```

The same trace tree shape used to debug a 2000-image reconciler fleet in
production — applied to one fridge.

### Live Elastic dashboard

[Elastic Cloud · My Observability Project](https://my-observability-project-d36723.kb.us-central1.gcp.elastic.cloud)

The "Fridge Reconciler" dashboard surfaces:
- reconciles ran · convergence rate · avg iterations · avg judge score
- judge-score timeline split by `fridge.generation` (the "ate the eggs" divergence)
- recipes table (titles · prep time · iterations · judge score)
- iteration count histogram
- p95 latency by span name

---

## Try the demo

```bash
# Edit fridge.yaml — pretend you ate the eggs.
sed -i '' '/name: eggs/{n;s/quantity: [0-9]*/quantity: 0/;}' fridge.yaml

# Commit + push.
git commit -am "ate the eggs"
git push
```

Within ~60–120 seconds:
- A Check Run named **`dev-mgreau-fridge-reconciler (fridge.yaml)`** appears on the commit.
- Open it — the recipe is rendered as Markdown in the summary.
- Hop to the Elastic dashboard — a new trace appears with a fresh generation hash,
  judge score, and full agent reasoning.

---

## What's stochastic

- Recipe content (the Chef invents different dishes for the same inputs across runs)
- Judge score per recipe (typically 0.25 – 0.95 spread on identical inputs)
- Iteration count (mostly 1, occasionally 2 or 3)

That stochasticity is the talk's central tension: **structural drift checks
(Critic) are fast and cheap; quality evaluation (Judge) is slow and expensive.
Production reconcilers need both layers.**

---
