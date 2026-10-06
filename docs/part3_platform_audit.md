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

### Section 1: Three highest-priority risks

- **Privileged access is not controlled tightly enough.** `sample_access.csv` lists a shared `admin` workspace-admin account without MFA (INC-009: P1 audit finding), a former employee with workspace-admin access, and a former employee still owning many jobs (`sample_jobs.csv`, e.g. J001-J004). This creates account misuse and operational continuity risks. 
    **First action:** 
    - disable stale human identities
    - remove the shared account
    - require MFA/SSO for administrators
    - transfer job ownership and credentials to governed service principals

- **Production ingestion can silently destroy or corrupt data.** INC-003 records an empty source result overwriting `raw_transfer`; INC-011 records schema drift turning values into NULL; INC-002 and INC-012 show silent drops and unreliable watermarking. These indicate missing data-quality gates, not isolated source issues. 
    **First action:** 
    - block destructive writes unless schema and row-count/freshness checks pass
    - quarantine unexpected input and alert before advancing watermarks

- **Critical jobs fail or disappear without timely detection or clear ownership.** J011's Meta upload fails because API v22 was deprecated, while INC-001 took 24 hours to detect; INC-008 took four days to detect a job missing from orchestration. J023 has an unknown owner, and J022 is failing on an expired PAT. 
    **First action:** 
    - inventory production jobs and dependencies
    - assign accountable owners
    - freshness/completion alerts
    - prioritize Meta API recovery and credential rotation.

### Section 2: Three cost or performance wins

- **Right-size the nightly sandbox.** J024 (`ds_sandbox_overnight`) costs **€3,200/month** and runs six hours every night on the large job cluster. Confirm usage with its owner, then move it to on-demand/manual or a smaller, bounded schedule; this is the largest obvious saving if overnight execution is not required.

- **Retire or modernize wasteful always-on clusters.** C07 costs **€540/month** and belongs to a former employee; C08 costs **€1,100/month**, never auto-terminates, and runs unsupported DBR 11.3. Confirm no dependencies, then terminate C07 and migrate C08 workloads to a supported job cluster with auto-termination. Potential avoidable spend: **€1,640/month**, plus reduced operational risk.

- **Optimize the known slow data path.** J028 (`credit_files_aggregation`) costs **€1,800/month**, takes 95 minutes, and notes an unoptimized table; INC-004 reports 80+ minute queries blocking dashboards. Profile the query, optimize/compact the table and review its partitioning, then compare runtime and dashboard latency before/after. This targets both compute duration and user impact.

### Section 3: AI-agent readiness

Make a **governed agent-serving boundary** the structural change before query load triples. `sample_access.csv` says all Claude/internal-agent queries use the shared `ai-agent-readonly` principal; `platform-svc` is a workspace admin with access to many catalogs and is also used by the agent query service. This prevents reliable per-agent attribution and gives agent traffic more reach than it needs.

Create a distinct service principal per agent/application, grant read-only access to approved curated Unity Catalog views, and apply row filters/column masks to sensitive data. Route queries through a dedicated SQL warehouse with agent/team workload tags, query timeouts, concurrency limits, and autoscaling. C06 (`sql_warehouse_xl`) currently backs the service continuously at **€4,800/month**; first establish per-agent query volume, latency, failures, and attributed cost, then size capacity from measured demand. Alert on access-denial spikes, abnormal query volume, latency, and budget thresholds.

### Section 4: Questions before committing

1. Which datasets are formally approved for agents, and what sensitive fields or row-level restrictions must be enforced?
2. Who owns each job, integration, and service principal—including the former employee's jobs—and what are the business-critical recovery/freshness SLAs?
3. What budget and latency targets should agent workloads meet, and can we change schedules or move workloads (for example J024) without disrupting users?
