# Fridge Reconciler — Demo Repo

A toy reconciler driven by the [DriftlessAF](https://github.com/driftlessaf/go-driftlessaf) framework.
Built for the Chainguard × Elastic meetup talk:
*"An open source agentic reconciliation framework proven at scale."*

## How it works

- **`fridge.yaml`** — observed state. What's actually in the fridge.
- **`goal.yaml`** — desired state. What we want for dinner.
- Pushing to either file triggers a deployed reconciler. A fleet of four agents
  (Inventory → Chef ⇄ Critic → Judge) figures out a recipe and posts it as a
  GitHub Check Run on the commit.

## Demo trigger

Edit `fridge.yaml` (e.g. set `eggs.quantity: 0` — "I ate the eggs"), commit, and
watch the Check Run appear within ~30 seconds with a new recipe.

Source code: TBD (private during the meetup).
