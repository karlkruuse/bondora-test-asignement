# Part 2: Reverse-ETL Pipeline Design

**Time budget: ~45 minutes. No code required, write your design directly in this file.**

## The scenario

Marketing wants to push audience and conversion data from our Databricks warehouse to four ad platforms: **Google Ads, Meta, Criteo, and CleverTap**. Today this happens via a tangle of one-off notebooks. We want to consolidate it into one framework.

**Scale and constraints:**

- 12 distinct audiences across 5 countries (EE, FI, DE, ES, NL). Sizes range from ~10K to ~1M users each.
- Daily refresh. Marketing expects audiences to be live in the ad platform by 9 AM local time.
- Each ad platform has its own API quirks: batch limits, async vs sync responses, rate limits, sometimes user-by-user error responses inside a batch.
- The source is Databricks. Audience definitions live in dbt models owned by the analytics engineers, you don't control those.
- Privacy team requires that we can prove which users were sent to which platform on which day. Audit retention: 13 months.
- The source dataset for some audiences is itself recomputed daily, so users can churn in and out of an audience. Yesterday's "send" is not today's truth.

## What we want from you

Write a 1-2 page design (you can use headings, diagrams in ASCII or mermaid, bullet points - whatever helps you explain). Cover at least these five questions. Take a position, don't list options without picking one.

### 1. Architecture

Sketch the components and how data flows from the dbt mart through to each ad platform. Where does code run (Databricks notebook? Job? External worker?). What's the orchestrator. Where does state live.

### 2. Idempotency

If today's run partially fails halfway through Google Ads at 8:14 AM and you re-run at 8:30, what happens? How do you avoid double-counting conversions, and how do you avoid skipping users? Be specific.

### 3. Diff vs full snapshot, and how you handle removals

The audience definition is recomputed daily, so users come in and out. Two related design questions:

a) Do you send the full audience snapshot to each platform daily, or only the diff (adds and removes) since yesterday? Pick one and justify.

b) Removal is harder than add. Most ad platform APIs need an explicit "remove this user from this audience" call - simply stopping to include them in future sends doesn't shrink the audience. How does your design track which user+platform pairs need a removal call on a given run? What if that removal call fails on one user but succeeds on the next ten thousand?

### 4. Observability

Three audiences need to know things about this pipeline. Design for all three:

- **You, on-call.** What metrics get emitted per run, what conditions trigger a page (be specific - "latency too high" isn't useful, give an actual threshold and what dashboard you check first).
- **Marketing.** They want to confirm "did today's run succeed, and how big was each audience" without messaging you. What do they see, where, and on what refresh cadence.
- **Future you, doing a postmortem.** A user complains "I should be in audience X but I'm not". What records do you need to be able to answer that within 5 minutes.

### 5. The trade-off you made

Every design has one painful trade-off. Tell us what yours is, what you chose, and what you'd reconsider if a constraint changed.

---

## Your design starts here

### 1. Architecture and assets

Use a **Databricks Workflow** as the daily orchestrator. It starts after dbt has completed, validates all 12 audience models, and fans out work by country and platform. Schedule each country's run using its local time zone and daylight-saving rules; start early enough to leave a retry window before the 9 AM local go-live expectation. Keep audience definitions in the analytics-owned dbt models; this pipeline reads their published outputs and does not modify them.

Run transformations and state management in Databricks. Use one reusable task implementation and platform-specific API adapters/workers for Google Ads, Meta, Criteo, and CleverTap. Adapters own API-specific batch limits, rate limits, async polling, and parsing of per-user results. Keep credentials in a secret manager and grant tasks only the access they need.

```text
dbt audience models (existing, persistent; analytics-owned)
                    |
                    v
Databricks Workflow (daily orchestration; country-local schedules)
                    |
                    v
[1. Validate and freeze today's inputs]
  Output: audience_snapshot
          PERSISTENT Delta table; immutable run snapshot
                    |
                    v
[2. Compare with confirmed platform state]
  Reads:  audience_snapshot + platform_membership
  Output: membership_changes
          SESSION-SCOPED Spark DataFrame, consumed within this task
                    |
                    v
[3. Persist planned actions]
  Writes: delivery_outbox
          PERSISTENT Delta table; retryable per-user actions
                    |
                    v
[4. Deliver pending actions through platform adapters]
  Calls:  Google Ads | Meta | Criteo | CleverTap
  Writes: delivery_audit      PERSISTENT; per-user attempt/outcome, 13 months
  Updates: delivery_outbox    PERSISTENT; status/retry data
  Updates: platform_membership PERSISTENT; confirmed membership only
                    |
                    +---- incomplete work remains in outbox for retry
                    +---- confirmed state is the baseline for tomorrow

```

**Persistent outputs and retention**

- `audience_snapshot` — Frozen copy of each run's desired audience, used to make retries and investigations reproducible; retain each run for **35 days**.
- `delivery_outbox` — Per-user platform actions and their processing/retry status; retain until every action is resolved, then for **35 additional days** for operational troubleshooting. Do not expire unresolved work automatically.
- `delivery_audit` — Append-only record of per-user delivery attempts and outcomes for privacy investigations; retain for **at least 13 months**.
- `platform_membership` — Latest confirmed user membership by audience and platform, used as the next run's comparison baseline; retain the current state **for as long as that audience-platform integration is active**, deleting it only through a controlled decommissioning process.


### 2. Idempotency and partial failures

Use **at-least-once delivery with per-user tracking**;  Each membership action has operation key such as `(run_date, audience, platform, user_token, action)`. The worker batches pending outbox records according to the destination's API limit, then records each user's individual result. Successful records are completed; failed records are retried with bounded exponential backoff; unknown outcomes remain explicitly unknown until reconciled or safely retried.

If Google Ads partially fails at 8:14 and the job restarts at 8:30, it resumes from pending/failed/unknown outbox rows, not from the beginning of the audience. It never advances a batch offset past unconfirmed users. 

If a call times out after the platform may have accepted it, query/reconcile its status before retrying; if that API offers no idempotency or lookup, document the duplicate risk and use its safest available operation semantics. For conversions, use the source event's immutable ID as the deduplication key and store it with the delivery outcome so a replay cannot count the event twice when the platform supports deduplication.

Write an audit attempt record for each call/result, and update `platform_membership` only after confirmed success. If the worker crashes between platform success and recording success, the stable key or reconciliation handles the replay.

### 3. Daily diffs and removals

Send **deltas (adds and removals)** daily, not the full audience. With audience sizes up to one million and four distinct APIs, this avoids repeatedly uploading unchanged members and helps meet rate limits. Run an initial full load when onboarding an audience/platform pair.

For each `(audience, platform)`, compare today's frozen desired members with `platform_membership`, which represents last **confirmed** membership:

- Desired now, not confirmed present: enqueue an `add`.
- Confirmed present, no longer desired: enqueue a `remove`.
- Otherwise: no membership action.

Persist every action in the outbox before calling the API. Record per-user results: if a removal batch succeeds for 9,990 users and fails for 10, the 9,990 are marked complete and only the 10 unsuccessful/unknown removals stay retryable. Do not update confirmed membership for failed or unknown outcomes. Periodically reconcile with platform membership exports/status APIs where available; where a platform cannot report membership, the confirmed ledger is our operational record and drift is a known limitation to monitor.

### 4. Observability and audit

Emit run-level and per-audience/country/platform metrics: snapshot size, adds/removes, attempted/succeeded/failed/unknown users, conversion events, retry count, API latency, rate-limit responses, oldest pending item, and completion time.

- **On-call:** page if any country/platform delivery is still incomplete at **8:45 AM local time**, or if more than **1% of attempted user actions remain failed/unknown after three retries**. Start at the dashboard showing deadline status, pending backlog, and failures grouped by platform/API response. Alert earlier when the measured processing rate predicts missing the 8:45 checkpoint.
- **Marketing:** provide a dashboard showing today's run status, audience sizes, last successful sync, and platform/country completion. Refresh it at least every five minutes while a run is active and show partial/in-progress status instead of a misleading overall success.
- **Postmortem / privacy audit:** append a record per user/action/attempt to `delivery_audit`, including a stable protected user token, audience, platform, run ID/date, add/remove/conversion action, attempt timestamp, result, and platform request/response reference where available. Retain it for at least **13 months**. Also retain run-to-source snapshot lineage so an investigation can tell whether the user was in the input, planned for delivery, attempted, and confirmed. Restrict and audit access; avoid raw identifiers in application logs. This enables answering “was this user selected, sent, and accepted by platform X on date Y?” within minutes.

### 5. Trade-off

The choice is **daily diffs plus durable per-user state**. This reduces API volume and makes removals/retries precise, but it adds storage, state transitions, and dependence on our confirmed-membership ledger. I would retain periodic reconciliation where platform capabilities allow and clearly surface when state is not independently verifiable. If a platform's API makes per-user outcomes or safe retries impractical, I would revisit the adapter strategy for that platform—potentially using a platform-supported full replacement workflow—rather than pretending the shared delta semantics are reliable there.
