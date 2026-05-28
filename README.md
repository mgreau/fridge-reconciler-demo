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
   ┌─── per agent execution ──────────────────────────────────────────┐
   │                                                                   │
   │   metaagent ─► Vertex AI (Claude)                                 │
   │       │                                                           │
   │       ├─ framework auto-emits OTel spans:                         │
   │       │     • invoke_agent                                        │
   │       │     • execute_tool submit_result                          │
   │       │       attrs:  gen_ai.input.messages   = full JSON         │
   │       │               driftlessaf.tool.reasoning = LLM text       │
   │       │               gen_ai.output.messages  = tool result       │
   │       │                                                           │
   │       └─ emits GenAI metrics:                                     │
   │             • gen_ai.client.token.usage                           │
   │             • genai.tool.calls                                    │
   │                                                                   │
   │   reconciler-owned spans (reconcile, iter-N, chef.propose,        │
   │   critic.evaluate, judge.score, inventory.parse) each stamp:      │
   │             • gen_ai.system           "fridge-reconciler"         │
   │             • gen_ai.operation.name   <span-name>                 │
   │             • fridge.*                <span-specific>             │
   │                                                                   │
   └─────────────────────────────────┬─────────────────────────────────┘
                                     │
                                     ▼  OTLP/HTTP  (Authorization: ApiKey ...)
   ┌──────────────────────────────────────────────────────────────────┐
   │                  Elastic Cloud Serverless APM                     │
   │                                                                   │
   │   service.name : mgreau85-fridge-recon-rec  (Cloud Run service)   │
   │                                                                   │
   │   Trace tree per reconcile:                                       │
   │     reconcile                       ← root span owned by us       │
   │      ├── inventory.parse                                          │
   │      ├── iter-1                                                   │
   │      │    ├── chef.propose                                        │
   │      │    │    └── invoke_agent                                   │
   │      │    │         └── execute_tool submit_result                │
   │      │    └── critic.evaluate                                     │
   │      │         └── invoke_agent                                   │
   │      │              └── execute_tool submit_result                │
   │      ├── iter-N …                                                 │
   │      └── judge.score                                              │
   │           └── invoke_agent                                        │
   │                └── execute_tool submit_result                     │
   │                                                                   │
   │   Attributes on the `reconcile` span (queried by dashboards):     │
   │     fridge.generation              SHA(goal + fridge + today)     │
   │     fridge.status                  CONVERGED / CANT_RECONCILE     │
   │     fridge.iterations_to_converge  1, 2, 3                        │
   │     fridge.judge_score             0.10 – 1.00                    │
   │     fridge.recipe_title            "Savory Yogurt Pancakes…"      │
   │     fridge.recipe_prep_minutes     20                              │
   │     fridge.recipe_ingredients      ingredient count                │
   │     fridge.final_drift_count       0 on CONVERGED                  │
   │     fridge.pantry_items            count                           │
   │                                                                   │
   │   Per-iteration attrs live on chef.propose / critic.evaluate:     │
   │     fridge.drift_count           total drift items                │
   │     fridge.drift_missing         missing ingredients              │
   │     fridge.drift_insufficient    quantity-too-low                 │
   │     fridge.drift_expired         used-but-expired                 │
   │                                                                   │
   │   Dashboard panels filter on:                                     │
   │     service.name == "mgreau85-fridge-recon-rec"                   │
   │     span.name    == "reconcile"      (one row per reconcile)      │
   │                                                                   │
   └──────────────────────────────────────────────────────────────────┘
```

> **Note.** The framework's `httpmetrics` overlays its own TracerProvider on top
> of ours, and its `llmSpanFilterProcessor` only forwards spans that carry at
> least one `gen_ai.*` attribute through to OTLP. Every span the reconciler
> emits therefore stamps `gen_ai.system = "fridge-reconciler"` so it survives
> the filter and reaches Elastic. Same trace tree shape used to debug a 2000-
> image reconciler fleet in production — applied to one fridge.

### Live Elastic dashboard

[Elastic Cloud · My Observability Project]([https://my-observability-project-d36723.kb.us-central1.gcp.elastic.cloud](https://my-observability-project-d36723.kb.us-central1.gcp.elastic.cloud/app/dashboards#/view/54225184-9fd9-4aac-b95c-01442817085c?_g=()))

The "Fridge Reconciler" dashboard surfaces:
- reconciles ran · convergence rate · avg iterations · avg judge score
- judge-score timeline split by `fridge.generation`
- recipes table (titles · prep time · iterations · judge score)
- iteration count histogram
- p95 latency by span name

<img width="2776" height="1400" alt="image" src="https://github.com/user-attachments/assets/2e89838d-6a63-4a62-88fb-2017f9794736" />

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
