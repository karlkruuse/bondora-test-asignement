# Part 3: Platform Audit

**Time budget: ~30 minutes. Written deliverable, no code required.**

## The scenario

You started yesterday. Today your manager hands you four exports from our Databricks workspace and says: **"The previous platform owner left two weeks ago. Tell me what worries you, what's expensive, and what we should fix first."**

The exports are in the `data/` directory:

- `sample_jobs.csv` - 30 scheduled jobs with owner, cluster, schedule, last status, cost
- `sample_clusters.csv` - 8 clusters with sizing and runtime info
- `sample_incidents.csv` - 12 production incidents from the past 6 months
- `sample_access.csv` - users and service principals with their access

The numbers are fictional but the patterns are realistic.

**Important context: AI agents have started querying our warehouse heavily over the last quarter, and we expect 3x query load from agents over the next 6 months.**

## What we want from you

A short audit report. ~1.5 pages, no longer. Use bullet points and short paragraphs - don't pad.

### Section 1: Three things that worry you most (risks)

Look across the four CSVs. Pick the three issues with highest blast radius. For each:

- What's the issue, in one sentence
- Why it matters (what's the failure mode)
- What you'd do about it (concrete first action)

### Section 2: Three cost or performance wins

Same format. Show your reasoning: which lines in which CSV led you there.

### Section 3: AI-agent readiness

Given that agent traffic is going to triple in 6 months, what one structural change to the platform would you make to be ready for it? Think beyond raw compute. What needs to be true about access, observability, query patterns, or cost attribution before agents become the dominant workload?

### Section 4: What you'd ask before committing

Three questions you'd ask your manager or a colleague before you'd actually start fixing anything. Tells us what context you know you're missing.

---

## Your audit starts here

> Write below this line.

