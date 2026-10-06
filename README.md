# Data Engineer Case Study

Welcome. This repo contains three exercises that mirror the kind of work you'll actually do on the team. Spend roughly **2 hours total**, no more than 3. We care more about how you think than polish.

## How to set up

You have two options. Both work, pick whichever you prefer.

**Option A: GitHub Codespaces (zero install).** Click `Code` -> `Codespaces` -> `Create codespace on main`. Wait \~2 minutes for the container to build. The container installs Java 17 for local PySpark. Open `notebooks/part1_ingestion_debug.ipynb`, pick the Python kernel, you're done. If you already have a codespace, rebuild its container to apply configuration changes (`Codespaces: Rebuild Container` from the command palette).

**Option B: Local with VS Code + Dev Containers.** Clone the repo, open in VS Code, accept the "Reopen in Container" prompt. Requires Docker.

If neither works for you, a plain local Python 3.11 + Java 17 + `pip install -r requirements.txt` is enough. The notebook uses local PySpark + Delta Lake, no Databricks account needed.

## What's in here

| Part | File | Time | Focus |
|---|---|---|---|
| 1 | `notebooks/part1_ingestion_debug.ipynb` | \~45 min | Debug a failing ingestion job, write a defensive wrapper |
| 2 | `docs/part2_reverse_etl_design.md` | \~45 min | Design a reverse-ETL pipeline (no code, written design) |
| 3 | `docs/part3_platform_audit.md` | \~30 min | Audit a fictional Databricks environment (`data/` directory has inputs) |

You can do the parts in any order. If you get stuck on one, move on - we'd rather see thoughtful partial answers across all three than perfection on one.

## How to submit

Push your work to a fork or branch and share the link. Or zip the repo and email it. Either is fine.

## Ground rules

- AI assistants are fine to use. We assume you will, and we'll ask you to walk through your work in a follow-up so use them as a tool, not a substitute for understanding.
- Don't over-engineer. A clear sketch beats half-finished perfection.
- If something is ambiguous, write down the assumption you made and proceed.

## Questions

Reply to the email thread or message your point of contact. No question is dumb.
