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

> Write below this line. Add headings, code blocks, diagrams as you like.

