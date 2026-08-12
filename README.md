# docs/ — Infrastructure Documentation

This folder holds the infra/DevOps documentation for the Primary Diagnosis
Validation project. These are living documents — update them as things change.

## What's here

| File | What it's for | Update when |
|---|---|---|
| `INFRA_STATE.md` | Master snapshot of the whole infra — the "paste into any AI/chat to restore context" file. Read this first. | Any resource added/changed; tasks completed |
| `config-reference.md` | Every secret and setting: name, where stored, who reads it, current value. The contract between infra and code. | A secret/setting is added, renamed, or an auth decision lands |
| `governance.md` | Security decisions and the reasoning behind them (why access-policy mode, why key-based auth, PHI boundary). | A security-relevant decision changes |
| `deployment-guide.md` | How to build the whole environment from scratch, step by step. | A build step changes; when wiring/images are done |
| `lessons-learned.md` | What differed from plan and why (Pinecone→Cosmos, RBAC wall, etc.). | Anytime something surprises you or you make a notable decision |

## The one habit that matters

Update these **in the moment**, not at the end. When you wire an env var, tick
it in config-reference right then. When an auth decision lands, update governance
that day. Five minutes each time beats reconstructing the whole story on
submission day — and since you lived every decision, keeping them current is just
noting what you already did.
