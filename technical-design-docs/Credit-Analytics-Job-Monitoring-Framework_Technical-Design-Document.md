# Credit Analytics Job Monitoring Framework: Technical Design Document

**Author:** Varun Duvvalla, Senior Data Scientist    [About Me](https://github.com/vduvalla/vduvalla/blob/ec14ac992fcd1b6523e5a9aaf2b9fd9ee0763208/about-me/README.md)   
**Organization:** PayPal Credit Org  
**Domain:** Consumer Credit and Payments, Data Platform Reliability  
**Scope:** Metadata-driven data-quality guardrail and orchestration library for batch analytics jobs, and a migration-validation harness used during the organization's Teradata-to-BigQuery migration  


---

> **Document Notice:** This document explains the methodology and system design at a high level. It does not include internal business data, system names, or confidential metrics. Product labels are used in a general way, and any accuracy figures are based on backtesting.


## Table of Contents

1. [Executive Summary](#1-executive-summary)  
2. [Problem Statement and Prior Art](#2-problem-statement-and-prior-art)  
3. [System Architecture](#3-system-architecture)  
4. [Testing Strategy and Design Decisions](#4-testing-strategy-and-design-decisions)  
5. [Operational Pipeline](#5-operational-pipeline)  
6. [Validation Methodology](#6-validation-methodology)  
7. [Results and Impact](#7-results-and-impact)  
8. [Lessons Learned and Extensions](#8-lessons-learned-and-extensions)

---

## 1. Executive Summary

This document covers the data quality and job orchestration framework that acts as a guardrail in front of downstream analytics loads for PayPal Credit Org's batch pipelines. The framework validates that upstream tables in the cloud data warehouse are fully loaded, correctly populated, and free of recurring quality defects before any downstream job is permitted to run. The same framework was also used as a validation harness during the organization's Teradata-to-BigQuery migration, where it ran paired record-count, aggregate, and enumeration checks between the legacy and target platforms to certify each migrated table before cutover.

Prior to this system, most daily and monthly analytics jobs executed blindly on whatever data happened to be present in the warehouse at execution time. Incomplete loads, late partitions, duplicate records, and out-of-whitelist enumeration values were discovered only after the fact, typically by stakeholders reviewing dashboards or by analysts catching anomalies in the ensuing week. Root-causing a bad dashboard or a miscalculated metric required hours of retrospective detective work across logs, upstream data, and alerting histories.

The framework tackles this by introducing a few design choices:

- **A metadata-driven test registry.** Every job is defined by a row (or small set of rows) in a single dependency table. The handler reads this configuration at run time and executes the configured tests without any code changes when a new job is added.  
- **A portfolio of purpose-built test types** (Baseline, lower-threshold, Metric, Metric-lower, Metric-external, cycle baseline, cycle metric, duplicate check, and a multi-day accepted-values sub-test) that cover the data-quality failure modes seen across the credit analytics stack.  
- **A polling-with-retry model** that distinguishes between "data is missing because it's late" (poll and retry) and "data is missing because something is wrong" (fail fast and alert), eliminating the flapping behavior of early prototypes that either gave up too quickly or retried forever.  
- **Day-delay enforcement as a core concept.** Each job declares how stale it is willing to tolerate, and the framework enforces that the target date is actually present in the table before considering the test passed. An explicit fallback mode is provided for legitimately sparse tables.  
- **Quantile-derived bounds learned from historical residuals.** Rather than fixed symmetric ±bands, the framework fits a quantile regressor on the residual distribution of each metric against its calendar and load context, and uses the learned conditional 5th-to-95th percentile envelope as the acceptance region. Bands widen on day-of-week and month-boundary conditions that are legitimately noisy and tighten on stable conditions, reducing both false positives and false negatives against the prior fixed-band design.  
- **Migration-validation mode.** During the Teradata-to-BigQuery migration, The same test harness reconciled paired tables across the two platforms. Row counts, aggregate sums on declared metric columns, and enumeration distributions on whitelist columns were compared side-by-side; discrepancies above the learned envelope were reported per day, per key, or per cycle, providing a signed, auditable certification that a migrated table matched its legacy source before downstream consumers were redirected.  
- **Idempotent job logging** to a central log with a STARTED → PASS/FAIL/Metrics Passed/Load Complete lifecycle, guarded against double-runs via a "last SUCCESS" pre-check that prevents the same daily job from re-running after it has already completed for the day.  
- **Alerting on failure** via the organization's standard notification channel, with per-day fail detail so the operator sees the exact offending dates, observed values, and violated bounds at the point of triage rather than after a ticket-driven investigation. A Slack-based alerting surface exists in the organization as a separate system; the job monitoring framework itself is responsible only for producing structured failure signals and is decoupled from any specific notification delivery mechanism.  
- **A single entrypoint** that every downstream notebook or scheduled job invokes as its first step. Success returns a structured result object and unlocks downstream execution; failure halts the containing notebook or DAG deterministically.

In production, the framework has reduced silent bad-data incidents to near zero across the monitored job portfolio, cut the time from data-defect occurrence to alert from hours to minutes, and made job onboarding a pure-configuration activity.

---

## 2. Problem Statement and Prior Art

### 2.1 The Data-Quality Gap

Batch analytics pipelines in a large enterprise warehouse present a set of data-quality failure modes that are distinct from the failure modes of application code:

1. **Partial-load invisibility.** A table exists, has rows for today, and appears loaded, but only a fraction of the expected volume is present because an upstream ETL failed mid-run. A downstream job that reads this table computes a correct-looking but materially wrong answer.

2. **Late-arriving partitions.** The target date for a daily job legitimately may not be in the table yet at the moment the downstream job starts. The correct behavior is to wait and re-check, not to fail. Distinguishing "late" from "broken" is a policy decision that the framework must encode.

3. **Silent enumeration drift.** A categorical column (product code, status code, segment flag) silently acquires a new value due to an upstream release. Joins, filters, and group-bys based on the historical value set still execute, still return rows, but the results no longer represent what the downstream logic assumes.

4. **Duplicate primary keys.** Two copies of the same upstream feed land in the same partition due to a retry or a double-schedule. A job that assumes row-level primary-key uniqueness over-counts by a factor of two. This defect is especially pernicious because aggregate totals often look "close enough" to escape eyeballing.

5. **Cycle-boundary incompleteness.** For credit portfolio data, the meaningful unit of work is a billing or reporting cycle, not a calendar day. Today's date may be fully loaded while this cycle is not, because the cycle hasn't ended yet. A cycle-aware guardrail must decide per cycle, not per date.

6. **Seasonality-biased range checks.** A naive "compare today to last 30 days" band check produces false positives around pay cycles, month-boundaries, and holidays. The framework needs a band-derivation rule that is reliable to the legitimately higher variance observed in this domain.

7. **Multi-day quality regression.** A subtle upstream bug can degrade the last several days of data without breaking the most recent day. A guardrail that only checks the most recent day will not catch this pattern. Multi-day window checks are required.

8. **Platform-migration risk.** During a large warehouse migration, every migrated table must be certified against its legacy source before any downstream consumer is redirected. Ad-hoc manual reconciliation of hundreds of tables is neither scalable nor auditable. The migration needed a harness that could execute a consistent reconciliation contract per table and produce a signed, dated pass/fail verdict.

### 2.2 Prior Approaches and Their Limitations

**No pre-check, trust the upstream (status quo ante).** The majority of analytics jobs executed whatever data was present. Failures were discovered downstream and were expensive to diagnose. This works fine when upstream is perfectly reliable. In practice, it is not.

**Per-job ad-hoc SQL checks embedded in notebook cells.** A small fraction of jobs had inline "sanity queries" at the top of the notebook comparing today's row count to yesterday's. This approach has three failure modes: each author writes their own check so the logic is inconsistent and often wrong; there is no retry policy, so a late load produces a false alarm; there is no logging, so the alert goes only to whoever happens to be watching the notebook output at that moment.

**Generic test frameworks.** Off-the-shelf data-quality test frameworks exist, but they did not natively express the polling-with-retry semantics required here. A straightforward "not null" or "row count" test fails on first run if the partition is simply late, with no retry loop, and does not distinguish "below the lower band because loading is incomplete" from "below the lower band because the pipeline is broken."

**Scheduler SLA-miss alerting.** Scheduler SLA features fire when a task exceeds its allotted wall-clock time, which is a necessary but insufficient signal. A task can complete on time against incomplete data. SLA misses tell us a task was slow; they do not tell us the data was wrong.

**Manual migration reconciliation.** For the warehouse migration specifically, the default practice would have been to compare legacy and migrated tables by hand, a sample query here, an aggregate check there, a spot inspection of enumerations. Given the scale of the migration (many tables, many downstream consumers) this was not scalable or auditable.

**what this framework adds.** nothing before this covered: (a) a uniform test library covering all observed failure modes, (b) polling-with-retry semantics that correctly distinguish late-but-arriving from broken, (c) a metadata-driven registration model that decouples test configuration from code, (d) centralized job logging with a "don't re-run if already succeeded today" guard, (e) quantile-derived conditional envelopes that adapt to the legitimate variance structure of each metric, and (f) a reusable migration-validation mode that applied the same contract to paired legacy-vs-target reconciliation.

---

## 3. System Architecture

### 3.1 High-Level Design

The framework is a single-module Python library organized into four logical layers:

```  
┌─────────────────────────────────────────────────────────────────────┐  
│                      ORCHESTRATION LAYER                            │  
│   Entrypoint · Dependency Read · Dispatch · Early-exit              │  
├─────────────────────────────────────────────────────────────────────┤  
│                      TEST EXECUTION LAYER                           │  
│ ┌──────────────┐ ┌───────────────┐ ┌──────────────┐ ┌─────────────┐ │  
│ │  Baseline /  │ │    Metric /   │ │   Cycle      │ │  Duplicate  │ │  
│ │ lower-thresh │ │ Metric-lower /│ │ baseline /   │ │   Check     │ │  
│ │ (polling)    │ │Metric-external│ │ cycle metric │ │             │ │  
│ │              │ │ (day-delay)   │ │ (per-cycle)  │ │             │ │  
│ └──────────────┘ └───────────────┘ └──────────────┘ └─────────────┘ │  
│   └─ Accepted-values sub-test   └─ Migration reconciliation mode    │  
├─────────────────────────────────────────────────────────────────────┤  
│                      LOGGING & ALERTING LAYER                       │  
│   Idempotent job log · Structured failure signal · Per-day detail   │  
│   STARTED / Metrics Passed / Load Complete / FAIL / SUCCESS         │  
├─────────────────────────────────────────────────────────────────────┤  
│                      INTEGRATION LAYER                              │  
│    Warehouse client · Notebook-namespace publication of cycle       │  
│    window · BI extract-refresh helpers                              │  
└─────────────────────────────────────────────────────────────────────┘  
```

### 3.2 Component Interaction

**Orchestration Layer.** A job invokes the library entrypoint as the first step of its containing notebook. The handler:

1. Initializes the run-state dictionary with a start timestamp, a test ledger, and sentinels for the cycle-window outputs.  
2. Prepares the warehouse execution environment (authentication, session configuration) through the organization's standard client. Implementation details of the warehouse client are encapsulated behind a thin internal wrapper so the rest of the framework is client-agnostic.  
3. Silently reads the dependency table for all rows where the job name matches and the active flag is set, ordered by dataset. A job may have multiple rows (one per table it guards); the handler iterates in order and short-circuits at the first failure.  
4. Applies defaults for optional columns so downstream test functions receive a normalized row regardless of dependency-table completeness.  
5. Silently queries the job log for the last SUCCESS date for this job. If the job is Daily and already succeeded today, or Monthly and already succeeded this month, the handler fails fast with a dedicated "already run" reason. This is a hard guard against double-runs from cron overlaps, manual retriggers, and workflow-engine replays.  
6. Inserts a STARTED record into the job log and enters the test-dispatch loop.

**Test Execution Layer.** Dispatch is driven by the test-type column on each dependency row. The handler routes to one of the following test functions:

| Test type | Purpose |  
|---|---|  
| Baseline | Two-sided band on total record count, with polling |  
| Lower-threshold | Lower-only band with polling |  
| Metric | Multi-day band on aggregated metric per day |  
| Metric-lower | Multi-day lower-only band on aggregated metric per day |  
| Metric-external | Same as Metric-lower, baseline read from a separately declared reference table |  
| Cycle baseline | Per-cycle band on record count vs. previous month |  
| Cycle metric | Per-cycle, per-metric band on aggregated metric vs. previous month |  
| Duplicate check | Multi-day primary-key uniqueness check |

Any metric-family test may additionally run a multi-day accepted-values sub-check if a whitelist and a target column are configured on the row. A migration-validation test variant is available for every test type: when the row declares a legacy-platform source, the test runs the same logic on both the legacy and target data and reports discrepancies beyond the learned envelope as failures.

**Logging & Alerting Layer.** Every state transition writes a row to the central job log. Inserts are quote-safe, single quotes in any reason string are doubled before write, to prevent both SQL syntax errors and any incidental injection risk from free-text failure reasons. On failure, the framework produces a structured failure signal with the job name, offending table, run time, reason, and dependency-table context; delivery to the operator's inbox or an alerting surface is the responsibility of the downstream notification system. A Slack-based alerting surface exists in the organization and consumes these signals, but it is a separate system and is not part of the monitoring library.

**Integration Layer.** Two integration behaviors worth noting:

1. **Notebook-namespace variables for cycle window.** Cycle tests compute the loaded-cycle window and publish the cycle window's bottom and top dates into the calling notebook's namespace. Downstream cells can read these variables directly without plumbing. Preexisting values are preserved if a cycle test does not run or does not compute a value, this prevents an upstream job that defines these variables from being silently clobbered by a no-op cycle test downstream.  
2. **BI extract-refresh helpers.** Thin wrappers around the organization's BI server client allow a notebook whose monitoring test has passed to kick off an extract refresh as its next step in the same process. These are not part of the monitoring flow proper, but they live in the same module because they share the "runs at the top of every analytics notebook" ergonomic role.

### 3.3 Metadata-Driven Design

Job-specific behavior is is driven entirely by a single row in the dependency table, not code.** The dependency table's columns describe, for each guarded table: the warehouse location, the date expression that identifies partitions, the staleness tolerance (day-delay), the lookback window, the test type, the metric column(s) if any, per-metric threshold(s) if any, a whitelist column and set for enumeration checks, an optional reference-table source for external baselines, a duplicate-check key expression, retry policy, frequency, an alert recipient, and an active flag.

Adding a new guarded job is a single row insert into this table. No library code changes are required, and no deploy is required. Changing a threshold, turning a job off during maintenance, or retargeting the alert recipient is likewise a single-row update.

### 3.4 Design Decisions at the Architecture Boundary

**Why a single module over a package with submodules?** The entire library is deployed as a single Python file. This was intentional. The target execution environment is the notebook runtime used by the analytics team, which imports the module at notebook start. A single file is copied and updated atomically; a multi-file package would require a full site-packages replacement to avoid mixed-version states during upgrades. The public API surface is a small number of functions, and splitting into submodules would increase complexity without reducing any measured operational burden.

**Why silent helpers for internal queries?** The library runs inside a notebook whose output is frequently pasted into tickets, dashboards, and discussion threads. Every SQL the handler issues would otherwise fill the notebook output with configuration queries that are noise relative to the test logic. Silent helpers suppress this noise; verbose helpers are used only for the actual test queries where the SQL and its results are the operator-facing signal. This is a intentional ergonomic choice to make the human-readable output of a run useful for diagnosis.

**Why a hard halt on failure rather than a return flag?** The handler is called at the top of a notebook or script. A return value would require the caller to check it on every invocation, and a forgotten check would allow a bad-data run to proceed. A hard-halt idiom stops both Jupyter kernels and Python scripts deterministically, so the guardrail cannot be silently bypassed by a missing conditional.

---

## 4. Testing Strategy and Design Decisions

### 4.1 Test Selection Philosophy

The test types grew out of real failure modes observed during rollout. Each one was added after a real failure mode slipped through. The framework does not try to cover every possible data-quality concern, it covers the ones that have actually occurred and have actually caused downstream harm. This keeps the configuration surface small and the behavior auditable.

All tests favor **conservative failure**: when the data could plausibly be late rather than broken, poll and retry; when the data is clearly out-of-bounds in a direction that cannot be explained by a late load, fail fast. This asymmetry reflects the operational reality that a false-positive alarm costs an on-call page, while a false-negative pass costs a corrupted downstream dataset and a multi-hour investigation.

### 4.2 Baseline and Lower-Threshold Tests

**Baseline** is a two-sided band check on total record count over the last `test_len_days` days, excluding the freshest `day_delay` days from baseline derivation (so the baseline is not contaminated by the partially loaded recent window). The acceptance band is the quantile-derived envelope described in Section 4.7. The decision rule is:

1. If the target date is missing from the result set, treat as "late" → poll. After the configured number of attempts without the target date arriving, fail. A fallback flag short-circuits this to a pass for legitimately sparse tables.  
2. If the latest record count is below the lower envelope bound, treat as "probably still loading" → poll. After the configured attempts still below, fail with the actual latest value and the required minimum.  
3. If the latest record count is above the upper envelope bound, fail fast. A value above the upper bound cannot be caused by a partial load, so retrying will not help; continuing to poll only delays the alert.  
4. If the latest record count is within the envelope, pass.

**Lower-threshold** is the lower-only variant for metrics where a high value is not a quality concern (e.g., counts that can legitimately spike upward on promotional dates). It uses the same polling and fallback semantics, but omits the upper-bound fast-fail.

**Why exclude the freshest rows from baseline derivation?** The freshest rows are the ones most likely to be partial. Including them in the baseline would pull the envelope downward and let partial loads pass. Excluding them means the envelope reflects historically complete days only, which is the right baseline for judging whether the latest day is itself complete.

### 4.3 Metric, Metric-Lower, and Metric-External Tests

The Metric family operates on aggregates of a specific column rather than total record count. Two differences from the Baseline test:

1. **Multi-day scope.** The test compares every day in the window against the envelope, not just the most recent day. A subtle upstream regression can degrade several days while leaving the most recent day intact. A single-day check would miss it.  
2. **Day-delay coverage enforcement.** The required target date must actually be present in the current table before the test runs. If missing, the test fails immediately. Unlike Baseline, no polling is performed at this step, the assumption is that metric tests gate downstream logic that reads the entire window, not just the latest day, so a missing target date means the window is malformed and retrying the same query will not change that fact. The fallback flag skips this enforcement for legitimately sparse tables.

The per-day decision varies by variant: Metric uses a two-sided envelope; Metric-lower uses the lower bound only; Metric-external uses the lower bound but reads the envelope from a separately declared reference source, supporting the pattern of evaluating a temp or current target against a stable prior-cycle baseline.

When days fail, the return reason is not just a count, it is a block of per-day fail lines listing each offending date, its observed value, and the violated bound. This block is embedded verbatim into the console summary and into the outbound failure signal (preserved in a monospace block by the notification consumer). The operator receiving the alert sees the exact offending days and values without opening a SQL console. The detail list is capped at a generous but finite line count with a "… and N more" tail to keep the alert readable.

### 4.4 Cycle Tests

Credit-portfolio data is organized around billing or reporting cycles. A calendar-day view is the wrong level of granularity for completeness checks because a cycle may legitimately span multiple days, and a cycle in-flight may have zero rows on some of its days and many on others.

The cycle tests address this by building a **per-cycle baseline from the previous month** and walking the current month's cycles in order, stopping at the first incomplete cycle.

**Cycle baseline** operates on record count per cycle:

1. Query the previous month for cycle counts grouped by cycle number. This yields one baseline row per cycle.  
2. For each cycle, fit the quantile envelope on the residual distribution of that cycle's count against its historical values.  
3. Query the current month for cycle counts in ascending cycle order.  
4. Walk the current-month cycles. For each one, if its count is within the envelope, mark it as fully loaded and advance. If not, stop and report the last fully loaded cycle.  
5. If at least one cycle was fully loaded, publish the cycle window as the `bottom_date` / `top_date` result, publish it into the notebook namespace, and pass.  
6. If no cycles are fully loaded, **hard fail**. Worth noting: the earlier behavior that converted this case to a pass under the fallback flag masked the most important failure mode for cycle-gated jobs, the upstream hasn't run yet, behind a flag that was typically set for the unrelated purpose of tolerating sparse tables.

**Cycle metric** is the same structure but over a list of metric aggregates. All metrics on a cycle must be within their respective envelopes for the cycle to count as fully loaded; the walk stops on the first metric failure.

**Why per-cycle rather than a single pooled baseline?** Volume varies materially by cycle number within a month. A pooled baseline would set the envelope at the average cycle volume, which would always fail small cycles and always pass large ones regardless of actual completeness. Per-cycle envelopes calibrate the check to the expected distribution of each specific cycle.

**Why stop at the first incomplete cycle?** Cycles load in order. If cycle N is incomplete, the assumption is that cycles N+1 and beyond (which have not started loading yet) are also not ready. The correct `top_date` is the end of cycle N-1, and the downstream job should operate on the resulting window. Continuing past the first incomplete cycle in the walk would include cycles that are either unstarted or in-flight, contaminating the downstream range.

### 4.5 Duplicate Check

The duplicate check test scans the window and fails if any combination of `(partition_date, key1, key2, ...)` has a count greater than one. The keys are configured on the dependency row; columns and simple expressions are both supported.

On failure, the test returns a reason that includes the number of impacted days and a bounded sample of offending key tuples. The sample cap keeps log inserts readable and keeps alert payloads digestible.

**Why scan multiple days rather than just today?** Duplicate introductions frequently manifest as a retroactive second load of a prior-day partition, not as a same-day double. A multi-day scan catches these backfills while they are still newly-introduced and quickly revertable. The window is bounded by the configured lookback to cap query cost.

**Why not enforce uniqueness as a primary-key constraint?** The target warehouse does not enforce primary keys at write time. The physical constraint must be tested rather than declared.

### 4.6 Accepted-Values Sub-Test

Any metric-family test can additionally run an accepted-values check if a whitelist and a target column are configured. The query scans the same multi-day window as the parent test and returns any rows where the target column's value is outside the configured whitelist. A nonempty result set fails the parent test.

**Why multi-day rather than single-day accepted values?** An earlier single-day design missed the case where an unaccepted value had leaked into the table several days ago and was still present. The multi-day scan catches the "new enumeration value silently appeared N days ago" pattern.

### 4.7 Quantile-Derived Bounds Learned from Historical Residuals

Earlier versions of the framework used symmetric multiplicative bands: `lower \= avg * (1 - t)`, `upper \= avg * (1 \+ t)`. This form has two real problems: First, a single scalar `t` applies uniformly across day-of-week, month-boundary, holiday, and campaign contexts, all of which have materially different legitimate variance. The band is either too narrow on high-variance contexts (false positives) or too wide on low-variance contexts (false negatives). Second, the symmetric form allows the same absolute deviation upward and downward, which is wrong for heavy-tailed metrics that can legitimately spike on promotional dates.

The current framework replaces the fixed symmetric band with a **conditional quantile envelope** learned from the historical residual distribution of each metric:

1. For each guarded (table, metric) pair, the framework maintains a history of daily residuals, where a residual is the observed value minus a local expectation (the trailing-window median for Baseline, the matched prior-cycle baseline for Metric, the same-cycle-number value for cycle tests).  
2. At model-fit time, a quantile regressor is trained on these residuals with features that include calendar context (day-of-week, day-of-month, month, holiday proximity), load context (day-delay position within the window), and, where available, campaign or cycle state.  
3. The regressor emits a conditional 5th and 95th percentile prediction for the residual distribution on each test day. The 90% envelope is computed as expectation \+ [q5, q95].  
4. A day passes if its residual falls inside the envelope; it fails otherwise, with the fail reason reporting the observed value, the expected value, and the envelope bounds.

The envelope is refit periodically as part of the same pipeline that runs the tests, so it adapts to structural shifts (new onboarding, promo cadence changes, seasonality drift) without manual adjustment. The quantile-regression form was chosen over parametric (Gaussian) intervals because the residual distribution is visibly heavy-tailed and asymmetric, and parametric intervals would understate the upper bound and overstate the lower bound.

**Why quantile regression over a pooled quantile?** A pooled quantile ignores the conditional structure, it treats a Saturday residual and a month-end residual as draws from the same distribution, when in practice they are not. The conditional quantile envelope widens on the contexts where residuals are legitimately larger and tightens on the contexts where they are legitimately smaller, which is exactly what we want.

**Backward compatibility.** A dependency-table row that declares a base threshold but does not declare a learned envelope falls back to the symmetric multiplicative form. This allowed a phased rollout and remains the default for very-low-volume tables where there is not enough residual history to fit the regressor reliably.

### 4.8 Migration-Validation Mode

During the Teradata-to-BigQuery migration, The same test harness reconciled paired tables across the two platforms. The mode is activated by declaring a legacy source on the dependency row; in that mode, each test runs twice, once against the legacy platform, once against the target platform, and compares the two results.

- **Baseline / Lower-threshold:** compare total record count per partition date; fail if any date's legacy-vs-target difference falls outside the learned envelope.  
- **Metric / Metric-lower / Metric-external:** compare the per-day aggregate; fail if any date's difference is outside the envelope, with per-day fail detail identifying which dates disagreed and by how much.  
- **Cycle baseline / Cycle metric:** compare per-cycle aggregates; fail if any cycle's legacy-vs-target difference is outside the envelope.  
- **Duplicate check:** run against both platforms; fail if either platform shows duplicates, or if the two platforms disagree on which keys duplicate.  
- **Accepted values:** compare the distinct-value sets on the target column; fail if either platform introduces a value the other does not have, or if the value-frequency distribution differs beyond the envelope.

Each migrated table's certification consisted of a signed, dated pass record written to the job log, with the reconciliation thresholds, windows, and results preserved for audit. Tables that did not reach a clean certification were sent back to the migration team with the exact per-day / per-cycle / per-key differences captured in the failure reason. This harness reduced the migration-reconciliation workstream from ad-hoc per-table SQL to a configuration-driven, auditable batch, and served as the prerequisite for cutover on each table.

---

## 5. Operational Pipeline

### 5.1 Pipeline Architecture

The full path of a single job invocation:

```  
Downstream Notebook / Scheduler  
             │  
             ▼  
┌────────────────────────┐  
│ Library entrypoint     │  
│   reset run state      │  
│   prepare warehouse env│  
└────────────────────────┘  
             │  
             ▼  
┌────────────────────────┐        ┌───────────────────────┐  
│ Read dependency rows   │───────▶│ No rows? → FAIL       │  
│ (silent)               │        └───────────────────────┘  
└────────────────────────┘  
             │  
             ▼  
┌────────────────────────┐        ┌───────────────────────┐  
│ Read last SUCCESS date │───────▶│ Already today? → FAIL │  
│ (silent)               │        │ Already month? → FAIL │  
└────────────────────────┘        └───────────────────────┘  
             │  
             ▼  
┌────────────────────────┐  
│ Insert STARTED         │  
└────────────────────────┘  
             │  
             ▼  
   ┌───── For each row ─────┐  
   │  Dispatch on test type │  
   │  Run test (verbose)    │  
   │  Record test result    │  
   └──────┬──────────┬──────┘  
          │          │  
         PASS       FAIL  
          │          │  
          ▼          ▼  
   ┌────────┐  ┌──────────────────────────────┐  
   │ Next   │  │ Finalize run · Print summary │  
   │ row    │  │ Insert FAIL (quote-safe)     │  
   └────────┘  │ Emit structured failure      │  
               │ Halt notebook/script         │  
               └──────────────────────────────┘  
          │  
          ▼  
┌────────────────────────┐  
│ All passed →           │  
│ Insert "Metrics Passed"│  
│  or "Load Complete"    │  
│ Publish cycle window   │  
│ Return result object   │  
└────────────────────────┘  
```

### 5.2 Job-Log State Machine

Every run writes a row to the central job log. The state machine is:

| Status | Written When | Triggers Downstream Alert |  
|---|---|---|  
| STARTED | Immediately after dependency read and last-success check | No |  
| FAIL | Any test failure, or "already ran today/month" | Yes |  
| Metrics Passed | All tests passed and at least one was a metric-family test | No |  
| Load Complete | All tests passed and none were metric-family | No |  
| SUCCESS | Only when the downstream job explicitly records its own completion | Configurable |

The library does **not** write SUCCESS on its own. The design assumption is that the downstream loader writes SUCCESS after its own logic completes; the library's terminal state is Metrics Passed or Load Complete, which encodes "the data was valid" rather than "the downstream job succeeded." This separation lets the guardrail gate the prerequisite without committing to the overall job's outcome.

**Quote-safe logging.** Failure reasons routinely contain punctuation, inequality symbols, and SQL fragments from duplicate samples. Before inserting into the log, the library replaces single quotes with doubled single quotes. This prevents both SQL syntax errors and any incidental SQL-injection risk from free-text reasons.

### 5.3 Polling Semantics

Only Baseline and Lower-threshold tests poll. The decision to poll is encoded in the test function, not the orchestrator, because the polling conditions are test-specific:

- Poll when the target date is not yet in the result set.  
- Poll when the latest count is below the lower envelope (plausibly still loading).  
- Do **not** poll when the latest count is above the upper envelope (cannot be explained by partial loading).  
- Do **not** poll for metric-family tests (day-delay coverage is enforced once, then the full multi-day window is scored deterministically).  
- Do **not** poll for cycle tests (cycle walking itself handles the "stop at incomplete" semantics).

Retry attempts and sleep time are both configurable per table. The operator tunes these per table based on observed upstream load timing, a table that typically lands within minutes of its SLA is configured with a small retry count and short sleep; a table that lands within hours uses a larger count and longer sleep.

### 5.4 Idempotence and Replay

The last-success pre-check is the primary idempotence guarantee. A Daily job that has a SUCCESS record for today will be refused on any further invocation. A Monthly job with a SUCCESS this calendar month is likewise refused. Worth knowing:

1. **A failed run does not consume the day's budget.** If today's run fails, tomorrow's run is unaffected, and the same day can be retried today until it succeeds. The guard is against *successful* reruns, not against retries of a failed run.  
2. **Manual overrides require either clearing the success record or waiting for the next period.** There is no force-rerun flag for a Daily job that has already succeeded today. In practice, if today's run succeeded, today's downstream data is already loaded, and rerunning would either be a no-op or would double-load.

### 5.5 Alerting Design

On failure, the library produces a structured failure signal, job name, failed table, run time, reason, originating dependency context, and hands it off to the organization's notification system. The library itself is delivery-agnostic; the downstream notification system (which, in the current organization, includes a Slack alerting surface) is responsible for routing, formatting, and retry.

A few design notes:

- **Structured, not free-text.** The failure signal carries typed fields (job name, table, run time, reason, dependency context) rather than a single rendered string. Downstream consumers can format for their own medium.  
- **Per-day fail detail is preserved verbatim.** When a multi-day metric fails, the per-day block (one line per offending date with observed value and violated bound) is part of the reason and is intended for monospace rendering. Notification surfaces that render monospace (most do) display the block correctly; those that do not fall back to a line-broken plain-text form.  
- **Delivery is best-effort.** The library's own responsibility ends at emitting the failure signal; if the downstream notification system is unreachable, the structured log record remains as the source of truth, and the hard halt on the caller guarantees the loudest signal (downstream job stopped) even if the notification path is misbehaving.

---

## 6. Validation Methodology

### 6.1 What "Passing" Means

A passing test is not a statement that the data is correct. It is a statement that the data does not exhibit any of the failure modes the framework is designed to detect, at the envelope learned for this (table, metric) pair. The distinction matters for two reasons:

1. **Envelopes are conditional.** The learned envelope widens on contexts where variance is legitimately large and tightens on contexts where it is small. A defect whose magnitude is within the envelope for its context will pass; downstream anomaly detection is responsible for catching subtler patterns.  
2. **Coverage is per-row.** A table is only guarded on the metrics and dimensions it declares in its dependency row. A column not listed as a metric is not checked, even if it is quietly broken. Scope discipline is the operator's responsibility.

The framework makes this distinction operational by reporting, on every run, exactly which tests ran and what bounds were checked. The run summary lists each test with PASS/FAIL and its details; a reviewer auditing a run can see the coverage explicitly.

### 6.2 Day-Delay Enforcement

Day-delay enforcement is the core validation that distinguishes "the data is there" from "the most recent day is there." For every metric-family test, the required target date must appear in the current-table result set. If missing and the fallback flag is not set, the test fails immediately:

```  
\<metric\>: target date \<required_date\> not loaded (day_delay=\<n\>)  
```

This is a hard failure, not a polled failure. The reasoning: metric tests evaluate the entire window, not just the most recent day. A missing target date means the test cannot assess the very day that the day-delay configuration claims should be ready. Polling that condition is equivalent to polling the clock, which is not useful.

The fallback flag exists for legitimately sparse tables, for example, reference tables that are only updated monthly but are referenced by daily jobs. In such cases the operator sets the fallback and the test scores against whatever days happen to be present.

### 6.3 Baseline Window Arithmetic

The lookback length and the day-delay parameters jointly define both the test window and the baseline derivation window:

```  
test window:        [today - lookback, today - day_delay]  
baseline material:  rows strictly older than (today - day_delay)  
```

The baseline is derived from historically older days so that the freshest days, which are by definition within the test window and may be partial, do not contaminate the baseline. This separation is the reason the same parameter appears both as a test-window boundary and as a baseline-exclusion offset.

### 6.4 Cycle-Boundary Reconciliation

For cycle tests, the validation unit is the cycle, not the day. A cycle is considered fully loaded if and only if:

- Its aggregate (count, or per-metric aggregate) falls within the per-cycle envelope derived from the previous month's same-cycle baseline.  
- All preceding cycles in the current month are also fully loaded by the same rule.

The second condition enforces contiguity. A scenario where cycle 3 is in-envelope but cycle 2 is not would stop the walk at cycle 2 and report the loaded window as ending at the end of cycle 1, not cycle 3. This is the correct behavior because cycle 3's in-envelope aggregate could be coincidental (a partial load that happens to resemble the prior-month baseline), and the downstream job reading the loaded window should not include a cycle whose contiguity with a known-incomplete predecessor has not been verified.

### 6.5 Migration-Mode Validation

In migration-validation mode, the validation question is not "is the data good?" but "does the target platform's data match the legacy platform's data?" The decision rule is symmetric: both platforms must produce the same result, within the envelope, for every scoring unit (day, cycle, key). Discrepancies are reported with both observed values, the envelope, and the signed delta. A table passes certification only when every scoring unit in the reconciliation window agrees within the envelope.

### 6.6 Result-Object Contract

Every run returns (or populates on exit) a result object with a stable shape:

- job identification and timing (name, start, end, duration)  
- cycle-window outputs (bottom and top dates where applicable, otherwise unset)  
- overall success flag and top-level message  
- per-test ledger: table, test type, status (PASS/FAIL), details string

Downstream orchestrators read this object to decide whether to proceed and, for cycle jobs, what date range to operate over. The shape is stable across versions; additive changes (new keys) have been permitted, but existing keys have not been renamed or retyped.

---

## 7. Results and Impact

### 7.1 Reliability Improvements

The framework's operational effect was assessed by comparing pre- and post-rollout incident counts for the guarded job portfolio:

| Metric | Before | After |  
|---|---|---|  
| Silent-bad-data incidents per quarter | Routinely multiple per month | Near zero in steady state |  
| Time-to-alert on a data defect | Hours to days (stakeholder discovery) | Minutes (synchronous failure at job start) |  
| Time-to-root-cause on a detected defect | Multi-hour log forensics | Immediate (failure reason includes offending table, metric, date, value, and envelope bound) |  
| Double-runs on Daily jobs | Occasional (cron overlap, manual reruns) | Zero (last-success guard) |  
| False-positive alarm rate | Noticeable under fixed symmetric bands | Materially reduced under learned envelopes |

Per-day fail detail in multi-day metric tests was a large driver of time-to-root-cause improvement: operators no longer have to re-run SQL to find which specific days violated which specific bounds.

### 7.2 Operational Efficiency

| Metric | Before | After |  
|---|---|---|  
| Onboarding a new guarded job | New inline SQL \+ manual review per job | Single row insert in the dependency table |  
| Adjusting a threshold | Edit and redeploy notebook | Row update in the dependency table |  
| Parking a job during maintenance | Comment out inline check, risk forgetting | Toggle the active flag on the dependency row |  
| Observability of currently guarded jobs | None (inline checks are invisible) | Single query against the dependency table |

The metadata-driven model moved operational changes from code changes to data changes, eliminating the deploy cycle for threshold tuning and on/off-boarding.

### 7.3 Migration Impact

During the Teradata-to-BigQuery migration, the framework's migration-validation mode was used to certify migrated tables across the portfolio. The effect:

- **Uniform reconciliation contract.** Every migrated table was certified against the same test contract (record count, aggregated metrics, enumerations, duplicates, cycle alignment), making the migration reviewable across tables rather than as a series of bespoke per-table reconciliations.  
- **Signed, dated certifications.** Each successful reconciliation produced a log record that could be referenced by the consumer teams before they redirected their reads to the target platform.  
- **Accelerated defect feedback.** Tables that did not reach clean certification were returned to the migration team with the specific per-day, per-cycle, or per-key differences captured in the failure reason, shortening the debug cycle from hours of manual triage to minutes.  
- **Reusability.** The same harness that served the migration continued to serve steady-state monitoring after cutover, with no code changes, only a dependency-row change to remove the legacy-source declaration.

### 7.4 Coverage Growth

The framework scaled from initial deployment on a single job to covering jobs across all the product lines in PayPal Credit Org, spanning daily, weekly, and monthly frequencies; the full portfolio of test types; and both legacy and target warehouse environments during migration. The configuration-driven architecture supported this growth without proportional engineering effort; each new job is a row.

---

## 8. Lessons Learned and Extensions

### 8.1 What Worked Well

**Metadata-driven over code-driven.** The decision to express every job as a dependency-table row paid off immediately. Adding or tuning a guarded job became a data operation, reviewable and auditable via SQL, schedulable by ops without engineering involvement, rollbackable by editing the same row. The complexity that would otherwise live in per-job notebooks lives in one place instead.

**Polling asymmetry.** The decision to poll only on conditions that can plausibly resolve (missing target date, below lower bound) and to fail fast on conditions that cannot (above upper bound) eliminated the "retry forever on a broken table" anti-pattern that earlier prototypes exhibited. In production, retries are used for genuinely late loads and not as a way to paper over real defects.

**Per-day fail detail.** Embedding the specific offending `(date, value, bound)` tuples directly into the failure reason, rather than linking to a log file, was the single highest-use usability improvement. Operators triage from the alert without opening any other tool.

**Learned envelopes over fixed bands.** The move from symmetric multiplicative bands to conditional quantile envelopes reduced both false positives on legitimately-variable contexts and false negatives on legitimately-stable ones. Operator trust in the guardrail improved noticeably once alerts stopped firing on known-benign day-of-week and month-boundary patterns.

**Hard fail on "no cycles loaded."** An earlier behavior that converted this case to a pass under the fallback flag was a design mistake. It masked the most important failure mode for cycle-gated jobs, the upstream hasn't run yet, behind a flag that was typically set for the unrelated purpose of tolerating sparse tables. Tightening to a hard fail restored the guardrail's core value proposition.

**Dual-use migration harness.** Designing the reconciliation mode as a core variant of every test (rather than a separate script) meant migration certification and steady-state monitoring shared the same code path. The operator who had learned how to read a steady-state failure reason could read a migration reconciliation reason without new training.

### 8.2 What I Would Do Differently

**Push-button envelope diagnostics.** Operators can inspect a learned envelope today, but only via library internals. A dedicated diagnostic command that prints the fitted envelope, the residual distribution, and the last N days' scoring outcomes would shorten onboarding and catch miscalibrated envelopes before they start paging on-call.

**Structured-log companion.** Today the reason column in the job log is free-text. A parallel structured schema (failed date, failed metric, observed value, lower bound, upper bound) would make it trivial to build portfolio-level views on failure patterns without string parsing. The free-text form is retained for human readability; adding a structured companion would serve the analytics use case without removing the human use case.

**Extract the silent query helpers as a shared utility.** These helpers are reused inside the module but are not exposed to callers. Other notebooks in the environment have reinvented similar wrappers. Exposing them as a public API would eliminate the duplication; the current module-scope underscore prefix was chosen to avoid commitment to a stable interface, but the pattern is now stable enough to publish.

### 8.3 Extensions and Future Directions

**Data-quality lineage integration.** When a guarded table fails, the failure should propagate as a signal to any job that depends on that table downstream, either by marking the downstream table as "upstream failed" in a lineage graph or by auto-skipping the downstream job. Currently, downstream jobs fail independently when they hit the same bad data; a lineage-aware system would suppress the second and third failure alerts when the first is already being investigated.

**Cross-job portfolio dashboard.** The job log contains every state transition for every guarded job. A portfolio-level dashboard summarizing daily pass/fail rates by job, failure-mode distribution, and MTTR (time from STARTED to next SUCCESS) would give ops a view the row-at-a-time log does not expose.

**Test-type extensibility via a registry.** The dispatch is currently an if/elif chain over test-type strings. A registry-plus-decorator pattern (each test function registers itself with its test-type key) would let new test types be added without editing the orchestrator.

**Re-use the harness for backfill validation.** The migration-validation mode reconciles a legacy platform against a target. The same mode, applied with "pre-backfill" and "post-backfill" as the two sides, would certify that a corrective backfill produced the expected deltas and did not introduce unintended side effects on dates it was not supposed to touch.

---

## Appendix A: Technology Stack

| Component | Role |  
|---|---|  
| Python 3 | Single-module library implementing orchestration, test execution, logging, and integration |  
| Cloud data warehouse | Execution substrate for test queries; both the legacy and target platforms supported during migration |  
| Notebook runtime | Host environment for the library; namespace-level integration for cycle-window publication |  
| Quantile regressor | Fits the conditional envelope on historical residuals for each guarded (table, metric) pair |  
| BI server client | Thin wrapper for extract refresh helpers invoked after passing runs |

## Appendix B: Dependency Configuration Reference

The handler reads a set of per-row columns that describe each guarded table. Columns are grouped by role rather than enumerated with physical names:

| Role | Purpose |  
|---|---|  
| Job identity | Logical job name matching the caller; active flag |  
| Target location | Dataset and table under test |  
| Date expression | Column or expression identifying the partition date |  
| Window parameters | Day-delay (staleness tolerance), lookback length |  
| Filter | SQL predicate appended to every test query |  
| Test type | Dispatch key; one of the test types enumerated in §3.2 |  
| Metric configuration | Metric column(s) and per-metric threshold(s), for metric-family tests |  
| Whitelist configuration | Target column and accepted-value set, for the accepted-values sub-test |  
| Reference source | Alternate baseline source for the external-baseline variant |  
| Duplicate key expression | Keys or expressions used by the duplicate-check test |  
| Retry policy | Attempt count and inter-attempt sleep |  
| Frequency | Daily, Weekly, or Monthly |  
| Fallback flag | Suppresses day-delay coverage enforcement for legitimately sparse tables |  
| Alerting target | Operator contact for failure notifications |  
| Migration pairing | Legacy-source declaration that allows migration-validation mode (if configured) |

Adding a new guarded job is a row insert; changing a threshold is a row update; parking a job during maintenance is an active-flag toggle. No library code changes are involved in any of these operations.

## Appendix C: Glossary

| Term | Definition |  
|---|---|  
| Baseline envelope | Conditional quantile envelope learned from the historical residual distribution of a (table, metric) pair; used as the acceptance region for all tests |  
| Day-delay | Operator-configured staleness tolerance; the target date a test evaluates is always the run date minus the day-delay |  
| Cycle | A billing or reporting cycle identified by a cycle number and cycle end date; the canonical unit for credit-portfolio data |  
| Fallback | A flag on the dependency row that suppresses day-delay coverage enforcement for legitimately sparse tables (does not apply to the "no cycles loaded" case) |  
| Job log | Centralized log recording every state transition (STARTED, FAIL, Metrics Passed, Load Complete, SUCCESS) for every guarded job |  
| Polling | Retry loop for plausibly-late loads; used only by Baseline and Lower-threshold |  
| Migration-validation mode | Variant of every test that reconciles paired legacy-vs-target data and reports differences beyond the learned envelope |  
| Quantile regressor | Estimator that outputs conditional quantiles (e.g., 5th and 95th percentiles) as a function of calendar and load context |  
| Quote-safe | Replacement of single quotes with doubled single quotes in free-text reasons before log inserts, preventing SQL syntax errors from punctuation in reasons |  
| Reference source | Alternate baseline source for the external-baseline variant |  
| Target date | The specific date a test gates on: the run date minus the day-delay |  
| Lookback | Length of the window used both for test scoring and for baseline derivation |  
| Cycle window (bottom / top date) | Cycle-test outputs identifying the fully-loaded data range; published into the notebook namespace for downstream use |

---

*This document describes a methodology and system architecture. No internal data, internal system identifiers, table names, project identifiers, or confidential performance metrics are disclosed.*  
