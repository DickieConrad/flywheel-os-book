# Chapter 5 --- Flywheel Health Metrics: `fact_flywheel_engagement`

> *"The flywheel's health is not measured by any single event. It is measured by rates of change --- whether the loop is accelerating, holding steady, or stalling. You cannot diagnose a flywheel by looking at one revolution."*

Chapter 4 designed the dimension that tracks *what* the personal model is at any point in time. This chapter designs the fact table that measures *how well* the flywheel is performing. Where `dim_personal_model` answers "what does the OS know about me?", `fact_flywheel_engagement` answers "is the OS getting better at knowing me --- or worse?"

The distinction matters because a flywheel can have a rich personal model and still be stalling. A user who granted broad observation permissions six months ago but has not expanded them since, whose prediction acceptance rate has plateaued, and whose action reversal rate is climbing --- that user's flywheel is decelerating, even though the dimension table looks healthy. Only a periodic snapshot of engagement metrics reveals the trajectory.

---

## 5.1 Why a Periodic Snapshot Fact

Ralph Kimball defines three types of fact tables, each suited to a different analytical question:

| Fact Type | Grain | Use Case | Example |
|-----------|-------|----------|---------|
| **Transactional** | One row per event | "What happened?" | `fact_agent_event` --- one row per EventBus event |
| **Periodic Snapshot** | One row per entity per period | "What is the state at regular intervals?" | `fact_session_snapshot` --- one row per session per hour |
| **Accumulating Snapshot** | One row per process lifetime | "How far along is this process?" | `fact_task_lifecycle` --- one row per task, updated at each milestone |

Chapter 2 introduced `fact_flywheel_stage` as a transactional fact --- one row per user per flywheel stage per execution event. That table captures *individual revolutions*: how long each stage took, how many entities were processed, whether the user approved an action. It is the raw event stream of the flywheel.

`fact_flywheel_engagement` is fundamentally different. It is a **periodic snapshot** --- one row per user per day --- that aggregates the transactional events into *rates of change*. The flywheel's health is not determined by any single observation, prediction, or action. It is determined by the trend: Is observation depth increasing? Is prediction accuracy improving? Is the user granting more permissions or revoking them? Are actions being completed or reversed?

These questions are unanswerable from a transactional fact table without expensive aggregation at query time. The periodic snapshot pre-computes the metrics that matter, enabling dashboard queries that scan a single row per user per day rather than thousands of transactional events.

### Why Daily?

The snapshot period is daily for three reasons:

1. **Flywheel dynamics operate on a daily cadence.** Users interact with their OS throughout the day; meaningful trends emerge across days, not minutes. A prediction that is wrong at 9 AM but correct by 3 PM (because the user corrected the model) should count as one learning cycle, not two separate accuracy events.

2. **Batch materialization aligns with daily processing.** Chapter 4 established that `dim_personal_model` materializes new SCD2 rows through nightly batch processing. The engagement snapshot runs on the same cadence, consuming the day's transactional events after they have settled.

3. **Storage efficiency.** One row per user per day keeps the table manageable. Even with 100,000 active users, a year of daily snapshots produces 36.5 million rows --- well within PostgreSQL's comfortable range with appropriate partitioning.

---

## 5.2 The `fact_flywheel_engagement` Table

### Grain Statement

> One row per user per snapshot period (daily). Each row captures the aggregated engagement metrics for a single user on a single calendar day, joined to the `dim_personal_model` version that was current at the end of that day.

### Column Reference

| Column | Type | Description |
|--------|------|-------------|
| `snapshot_key` | `BIGSERIAL` | Surrogate key (auto-increment). Primary key. |
| `snapshot_date_key` | `INTEGER` | Foreign key to `dim_date`. The calendar day this snapshot covers. |
| `model_key` | `BIGINT` | Foreign key to `dim_personal_model`. The SCD2 version that was current at snapshot time. Enables point-in-time correlation between engagement metrics and model state. |
| `user_id` | `UUID` | Degenerate dimension. The user whose flywheel this row measures. Denormalized from `dim_personal_model` for query convenience --- avoids a join for the most common filter. |
| `observation_depth` | `INTEGER` | Total observations processed this period. Measures how much raw data the flywheel ingested --- files accessed, communications observed, calendar events parsed. Higher is better: a deep observation surface feeds richer inference. |
| `prediction_count` | `INTEGER` | Total predictions made this period. Includes suggestions surfaced, context pre-fetched, and actions anticipated. A healthy flywheel makes more predictions as the model matures. |
| `prediction_accuracy` | `NUMERIC(5,4)` | Percentage of predictions the user accepted, acted on, or did not dismiss. Range 0.0000 to 1.0000. The most important single KPI --- it measures whether the model *understands* the user. |
| `action_count` | `INTEGER` | Total autonomous actions taken this period. Actions at any approval tier: suggested, confirmed, and fully autonomous. |
| `action_completion_rate` | `NUMERIC(5,4)` | Percentage of actions that completed successfully without user intervention. Range 0.0000 to 1.0000. Measures operational reliability of the action layer. |
| `action_reversal_rate` | `NUMERIC(5,4)` | Percentage of completed actions the user explicitly undone. Range 0.0000 to 1.0000. An inverse trust signal --- rising reversals indicate the model is acting against user intent. |
| `trust_score_delta` | `SMALLINT` | Net change in `trust_level` (from `dim_personal_model`) during this period. Positive values indicate trust growth; negative values indicate trust erosion. Zero means stable trust. |
| `engagement_velocity` | `NUMERIC(8,4)` | Composite flywheel speed metric (see Section 5.3 for formula). Combines observation depth, prediction accuracy, action completion, and trust movement into a single scalar. Higher is faster; negative means the flywheel is actively decelerating. |
| `observation_surface_expansion` | `BOOLEAN` | Did the user grant new observation permissions during this period? `TRUE` if the `observation_surface_scope` JSONB in `dim_personal_model` added new keys or expanded existing scopes. The strongest trust signal available. |
| `tenant_id` | `TEXT` | Tenant scope for Row-Level Security. Follows ORCHA's multi-tenant isolation pattern. |
| `created_at` | `TIMESTAMPTZ` | Row insertion timestamp. |

### Measure Classification

| Measure | Kimball Type | Rationale |
|---------|-------------|-----------|
| `observation_depth` | Additive | Observations from different users on different days can be summed meaningfully. |
| `prediction_count` | Additive | Predictions from different users on different days can be summed. |
| `prediction_accuracy` | Non-additive | A percentage cannot be summed across users or time. Must be averaged or recalculated from components. |
| `action_count` | Additive | Actions can be summed across users and time. |
| `action_completion_rate` | Non-additive | Percentage; requires weighted average. |
| `action_reversal_rate` | Non-additive | Percentage; requires weighted average. |
| `trust_score_delta` | Semi-additive | Additive across time (sum of deltas = total change) but not across users (summing trust deltas of different users is meaningless). |
| `engagement_velocity` | Non-additive | Composite metric; cannot be summed or averaged without decomposition. |

---

## 5.3 Flywheel KPIs

Each column in `fact_flywheel_engagement` maps to a Key Performance Indicator that diagnoses a specific aspect of flywheel health. Together, they form a complete picture of whether the loop is accelerating or stalling.

### Observation Depth

**What it measures:** How much the flywheel can *see*.

Observation depth is the count of discrete observations processed during the snapshot period. An observation is any data point the system ingested from the user's declared observation surface: a file opened, an email received, a calendar event created, a browsing session recorded, a location change detected.

**Why it matters:** Observation depth is the flywheel's fuel. Without observations, the inference layer has nothing to work with, predictions cannot be made, and actions cannot be taken. A declining observation depth --- even if the observation surface permissions have not changed --- signals that the user is migrating activity to contexts the OS cannot see (e.g., using a different device, switching to tools outside the observation API).

**Healthy range:** Depends on the user's observation surface scope. A user who has granted email, calendar, and file observation should produce 50--200 observations per day. A user who has only granted calendar observation might produce 5--20. The absolute number matters less than the *trend*.

### Prediction Accuracy

**What it measures:** How well the model *understands*.

Prediction accuracy is the ratio of predictions the user accepted or acted on to total predictions surfaced. A prediction counts as "accurate" if:
- The user explicitly accepted the suggestion
- The user opened the pre-fetched content
- The user did not dismiss the notification within 30 seconds (implicit acceptance)

A prediction counts as "inaccurate" if:
- The user explicitly dismissed it
- The user marked it as irrelevant
- The pre-fetched content was never accessed

**Why it matters:** Prediction accuracy is the single best proxy for model quality. A model that makes accurate predictions has correctly learned the user's patterns, preferences, and context. A model that makes inaccurate predictions is either working from insufficient observations or has learned the wrong patterns.

**Healthy range:** Early flywheel (first 30 days): 0.40--0.60. Mature flywheel (90+ days): 0.70--0.85. Exceeding 0.90 is rare and may indicate the system is only making "safe" (obvious) predictions rather than genuinely useful ones.

### Action Completion Rate

**What it measures:** How much the user *trusts* the action layer.

Action completion rate is the ratio of autonomous actions that completed successfully to total actions initiated. An action "completes successfully" if it reaches its intended end state without user intervention, reversal, or system error.

**Why it matters:** Completion rate bridges the gap between prediction and execution. The system may correctly predict that the user wants an email drafted, but if the drafted email requires extensive edits before sending, the action did not truly complete. High completion rates mean the action layer is executing with fidelity to user intent.

**Healthy range:** Stage 4 (Act with Approval): 0.80--0.95. Stage 5 (Autonomous): 0.90--0.99. Below 0.80 at any stage suggests the action layer is acting prematurely --- it has been granted autonomy that its accuracy does not support.

### Action Reversal Rate

**What it measures:** The *inverse* of trust.

Action reversal rate is the ratio of completed actions that the user explicitly undid to total completed actions. This is distinct from actions that failed to complete --- reversals are actions that *succeeded* from the system's perspective but that the user judged to be wrong.

**Why it matters:** Reversals are the strongest negative signal in the flywheel. A user who reverses an action is saying: "You did something I did not want." One reversal erodes more trust than ten correct actions build. Rising reversal rates are the earliest warning of flywheel stall.

**Healthy range:** Below 0.05 (5%) at all stages. Above 0.10 (10%) is a critical alert --- the system should automatically reduce autonomy stage. Above 0.20 (20%) means the flywheel has effectively stalled; the user no longer trusts the action layer.

### Trust Score Delta

**What it measures:** Direct trust movement.

Trust score delta is the net change in the user's `trust_level` (from `dim_personal_model`) during the snapshot period. It is the difference between the trust level at the end of the day and the trust level at the start.

**Why it matters:** While prediction accuracy and reversal rate are *proxy* signals for trust, the trust score delta is the *direct* measurement. It reflects the combined effect of all interactions --- good predictions, bad actions, surface expansions, surface contractions --- distilled into a single number.

**Interpretation:**
- **Positive delta (+1 to +5):** Flywheel is earning trust. Normal during early adoption and after successful autonomy demonstrations.
- **Zero delta:** Stable state. Healthy for mature flywheels that have reached equilibrium.
- **Negative delta (-1 to -5):** Trust is eroding. Requires investigation: check reversal rate, prediction accuracy, and any recent incidents.
- **Large negative delta (< -5):** Trust crisis. Likely caused by a privacy incident, a catastrophic wrong action, or a credibility failure (Apple Intelligence pattern).

### Engagement Velocity

**What it measures:** Composite flywheel speed.

Engagement velocity is a formula that combines the other KPIs into a single scalar representing overall flywheel health. It is designed to be:
- **Positive** when the flywheel is accelerating
- **Zero** when the flywheel is in equilibrium
- **Negative** when the flywheel is decelerating

**Formula:**

```
engagement_velocity = (
    (prediction_accuracy * 40)
    + (action_completion_rate * 25)
    - (action_reversal_rate * 50)
    + (trust_score_delta * 10)
    + (observation_surface_expansion::int * 15)
    - 30  -- baseline offset: velocity is 0 at "average" engagement
)
```

**Weight rationale:**
- Prediction accuracy (weight 40): The core flywheel metric. Accurate predictions drive everything else.
- Action completion (weight 25): Reliable execution builds trust.
- Action reversal (weight -50): Reversals destroy trust disproportionately. The negative weight is double the positive weight of completion because trust destruction is faster than trust building.
- Trust score delta (weight 10): Direct trust movement amplifies velocity.
- Observation surface expansion (weight 15): The strongest positive signal --- the user is actively choosing to feed more data into the flywheel.
- Baseline offset (-30): Centers the scale so that "average" performance produces a velocity near zero.

**Healthy range:** Accelerating: +5 to +40. Steady: -5 to +5. Stalling: below -5.

---

## 5.4 PostgreSQL DDL

The following DDL is compatible with Supabase (PostgreSQL 15+) and follows ORCHA's existing migration patterns established in `supabase/migrations/`.

```sql
-- =============================================================================
-- fact_flywheel_engagement: Periodic Snapshot Fact for Flywheel Health
-- =============================================================================
-- Measures the acceleration or deceleration of the personal data flywheel.
-- Aggregated daily from transactional events (fact_flywheel_stage) and
-- personal model state (dim_personal_model).
--
-- Grain: One row per user per calendar day.
-- Snapshot period: Daily (materialized by nightly batch).
-- Fact type: Periodic snapshot.
--
-- Joins:
--   snapshot_date_key -> dim_date.date_key
--   model_key -> dim_personal_model.model_key (SCD2 version current at EOD)
-- =============================================================================

CREATE TABLE IF NOT EXISTS fact_flywheel_engagement (
    -- ── Surrogate key ──────────────────────────────────────────────────
    snapshot_key            BIGSERIAL PRIMARY KEY,

    -- ── Dimension foreign keys ─────────────────────────────────────────
    snapshot_date_key       INTEGER NOT NULL,
    model_key               BIGINT NOT NULL,

    -- ── Degenerate dimension ───────────────────────────────────────────
    user_id                 UUID NOT NULL,

    -- ── Additive measures ──────────────────────────────────────────────
    observation_depth       INTEGER NOT NULL DEFAULT 0
                            CHECK (observation_depth >= 0),
    prediction_count        INTEGER NOT NULL DEFAULT 0
                            CHECK (prediction_count >= 0),
    action_count            INTEGER NOT NULL DEFAULT 0
                            CHECK (action_count >= 0),

    -- ── Non-additive measures (rates / percentages) ────────────────────
    prediction_accuracy     NUMERIC(5,4) NOT NULL DEFAULT 0.0000
                            CHECK (prediction_accuracy BETWEEN 0.0000 AND 1.0000),
    action_completion_rate  NUMERIC(5,4) NOT NULL DEFAULT 0.0000
                            CHECK (action_completion_rate BETWEEN 0.0000 AND 1.0000),
    action_reversal_rate    NUMERIC(5,4) NOT NULL DEFAULT 0.0000
                            CHECK (action_reversal_rate BETWEEN 0.0000 AND 1.0000),

    -- ── Semi-additive measures ─────────────────────────────────────────
    trust_score_delta       SMALLINT NOT NULL DEFAULT 0,

    -- ── Composite metric ───────────────────────────────────────────────
    engagement_velocity     NUMERIC(8,4) NOT NULL DEFAULT 0.0000,

    -- ── Boolean flags ──────────────────────────────────────────────────
    observation_surface_expansion  BOOLEAN NOT NULL DEFAULT FALSE,

    -- ── Multi-tenant isolation (ORCHA pattern) ─────────────────────────
    tenant_id               TEXT NOT NULL,

    -- ── Metadata ───────────────────────────────────────────────────────
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- ── Referential integrity ──────────────────────────────────────────
    CONSTRAINT fk_engagement_date
        FOREIGN KEY (snapshot_date_key) REFERENCES dim_date(date_key),
    CONSTRAINT fk_engagement_model
        FOREIGN KEY (model_key) REFERENCES dim_personal_model(model_key),

    -- ── Business rules ─────────────────────────────────────────────────
    CONSTRAINT uq_user_date
        UNIQUE (user_id, snapshot_date_key),
    CONSTRAINT chk_reversal_coherence
        CHECK (action_count > 0 OR action_reversal_rate = 0.0000),
    CONSTRAINT chk_completion_coherence
        CHECK (action_count > 0 OR action_completion_rate = 0.0000),
    CONSTRAINT chk_prediction_coherence
        CHECK (prediction_count > 0 OR prediction_accuracy = 0.0000)
);

-- =============================================================================
-- Indexes
-- =============================================================================

-- Primary access pattern: single user's flywheel health over time
CREATE INDEX idx_engagement_user_date
    ON fact_flywheel_engagement (user_id, snapshot_date_key DESC);

-- Dashboard pattern: all users on a specific date (daily health report)
CREATE INDEX idx_engagement_date
    ON fact_flywheel_engagement (snapshot_date_key);

-- Stall detection: find users with negative velocity
CREATE INDEX idx_engagement_velocity
    ON fact_flywheel_engagement (engagement_velocity)
    WHERE engagement_velocity < -5.0;

-- Tenant isolation
CREATE INDEX idx_engagement_tenant
    ON fact_flywheel_engagement (tenant_id, snapshot_date_key DESC);

-- Model version correlation: join to dim_personal_model
CREATE INDEX idx_engagement_model
    ON fact_flywheel_engagement (model_key);

-- Observation surface expansion events (rare but high-signal)
CREATE INDEX idx_engagement_surface_expansion
    ON fact_flywheel_engagement (user_id, snapshot_date_key)
    WHERE observation_surface_expansion = TRUE;

-- =============================================================================
-- Row-Level Security (Supabase pattern)
-- =============================================================================

ALTER TABLE fact_flywheel_engagement ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON fact_flywheel_engagement
    USING (tenant_id = current_setting('app.tenant_id', TRUE));

-- =============================================================================
-- Comments
-- =============================================================================

COMMENT ON TABLE fact_flywheel_engagement IS
    'Periodic snapshot fact measuring flywheel acceleration or deceleration. '
    'One row per user per day. Aggregated from transactional flywheel events '
    'and personal model state. Primary table for flywheel health dashboards.';

COMMENT ON COLUMN fact_flywheel_engagement.snapshot_key IS
    'Surrogate key. Auto-incrementing. Uniquely identifies each daily snapshot row.';
COMMENT ON COLUMN fact_flywheel_engagement.snapshot_date_key IS
    'FK to dim_date. The calendar day this snapshot covers.';
COMMENT ON COLUMN fact_flywheel_engagement.model_key IS
    'FK to dim_personal_model. The SCD2 version current at end-of-day.';
COMMENT ON COLUMN fact_flywheel_engagement.user_id IS
    'Degenerate dimension. User UUID denormalized for query convenience.';
COMMENT ON COLUMN fact_flywheel_engagement.observation_depth IS
    'Total observations ingested this day. Additive across users and time.';
COMMENT ON COLUMN fact_flywheel_engagement.prediction_accuracy IS
    'Ratio of accepted predictions to total predictions. Non-additive (0.0-1.0).';
COMMENT ON COLUMN fact_flywheel_engagement.action_reversal_rate IS
    'Ratio of reversed actions to completed actions. Inverse trust signal (0.0-1.0).';
COMMENT ON COLUMN fact_flywheel_engagement.trust_score_delta IS
    'Net change in trust_level during this day. Semi-additive over time.';
COMMENT ON COLUMN fact_flywheel_engagement.engagement_velocity IS
    'Composite flywheel speed. Positive=accelerating, negative=decelerating.';
COMMENT ON COLUMN fact_flywheel_engagement.observation_surface_expansion IS
    'TRUE if user granted new observation permissions during this day.';
```

---

## 5.5 Query Examples

### Query 1: Daily Flywheel Health Dashboard

The primary operational query. Returns a single-day snapshot of flywheel health for all users in a tenant, with human-readable labels.

```sql
SELECT
    d.calendar_date,
    fe.user_id,
    pm.autonomy_stage,
    CASE pm.autonomy_stage
        WHEN 1 THEN 'Observe'
        WHEN 2 THEN 'Suggest'
        WHEN 3 THEN 'Predict'
        WHEN 4 THEN 'Act with Approval'
        WHEN 5 THEN 'Autonomous'
    END AS stage_name,
    fe.observation_depth,
    fe.prediction_count,
    ROUND(fe.prediction_accuracy * 100, 1) AS accuracy_pct,
    fe.action_count,
    ROUND(fe.action_completion_rate * 100, 1) AS completion_pct,
    ROUND(fe.action_reversal_rate * 100, 1) AS reversal_pct,
    fe.trust_score_delta,
    ROUND(fe.engagement_velocity, 2) AS velocity,
    fe.observation_surface_expansion AS surface_expanded,
    CASE
        WHEN fe.engagement_velocity > 5 THEN 'ACCELERATING'
        WHEN fe.engagement_velocity >= -5 THEN 'STEADY'
        ELSE 'STALLING'
    END AS flywheel_state
FROM fact_flywheel_engagement fe
JOIN dim_date d ON fe.snapshot_date_key = d.date_key
JOIN dim_personal_model pm ON fe.model_key = pm.model_key
WHERE d.calendar_date = CURRENT_DATE - INTERVAL '1 day'
ORDER BY fe.engagement_velocity DESC;
```

### Query 2: Trust Trajectory Over 30 Days

Tracks a single user's trust evolution, showing the cumulative effect of daily trust deltas and the relationship between velocity and trust movement.

```sql
SELECT
    d.calendar_date,
    fe.trust_score_delta,
    SUM(fe.trust_score_delta) OVER (
        ORDER BY d.calendar_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_trust_change,
    pm.trust_level AS absolute_trust_level,
    ROUND(fe.engagement_velocity, 2) AS velocity,
    ROUND(fe.prediction_accuracy * 100, 1) AS accuracy_pct,
    ROUND(fe.action_reversal_rate * 100, 1) AS reversal_pct
FROM fact_flywheel_engagement fe
JOIN dim_date d ON fe.snapshot_date_key = d.date_key
JOIN dim_personal_model pm ON fe.model_key = pm.model_key
WHERE fe.user_id = :user_id
  AND d.calendar_date >= CURRENT_DATE - INTERVAL '30 days'
ORDER BY d.calendar_date;
```

### Query 3: Flywheel Stall Detection

Identifies users whose flywheel is decelerating --- engagement velocity has been negative for three or more consecutive days. This is the early warning system for flywheel stall.

```sql
WITH daily_velocity AS (
    SELECT
        fe.user_id,
        d.calendar_date,
        fe.engagement_velocity,
        -- Count consecutive negative-velocity days using the
        -- "gaps and islands" technique
        ROW_NUMBER() OVER (PARTITION BY fe.user_id ORDER BY d.calendar_date)
        - ROW_NUMBER() OVER (
            PARTITION BY fe.user_id, (fe.engagement_velocity < -5)::int
            ORDER BY d.calendar_date
          ) AS stall_group
    FROM fact_flywheel_engagement fe
    JOIN dim_date d ON fe.snapshot_date_key = d.date_key
    WHERE d.calendar_date >= CURRENT_DATE - INTERVAL '14 days'
      AND fe.engagement_velocity < -5
),
stall_streaks AS (
    SELECT
        user_id,
        MIN(calendar_date) AS stall_started,
        MAX(calendar_date) AS stall_latest,
        COUNT(*) AS consecutive_stall_days,
        ROUND(AVG(engagement_velocity), 2) AS avg_velocity
    FROM daily_velocity
    GROUP BY user_id, stall_group
    HAVING COUNT(*) >= 3
)
SELECT
    ss.*,
    pm.autonomy_stage,
    pm.trust_level
FROM stall_streaks ss
JOIN dim_personal_model pm
    ON pm.user_id = ss.user_id AND pm.is_current = TRUE
ORDER BY ss.consecutive_stall_days DESC, ss.avg_velocity ASC;
```

### Query 4: Observation Surface Correlation with Accuracy

Tests the hypothesis that expanding the observation surface leads to improved prediction accuracy. Compares accuracy before and after surface expansion events.

```sql
WITH expansion_events AS (
    SELECT
        fe.user_id,
        d.calendar_date AS expansion_date,
        ROW_NUMBER() OVER (PARTITION BY fe.user_id ORDER BY d.calendar_date) AS expansion_num
    FROM fact_flywheel_engagement fe
    JOIN dim_date d ON fe.snapshot_date_key = d.date_key
    WHERE fe.observation_surface_expansion = TRUE
),
before_after AS (
    SELECT
        ee.user_id,
        ee.expansion_date,
        -- Average accuracy in the 7 days BEFORE expansion
        (SELECT ROUND(AVG(fe2.prediction_accuracy) * 100, 1)
         FROM fact_flywheel_engagement fe2
         JOIN dim_date d2 ON fe2.snapshot_date_key = d2.date_key
         WHERE fe2.user_id = ee.user_id
           AND d2.calendar_date BETWEEN ee.expansion_date - INTERVAL '7 days'
                                     AND ee.expansion_date - INTERVAL '1 day'
        ) AS accuracy_before,
        -- Average accuracy in the 7 days AFTER expansion
        (SELECT ROUND(AVG(fe3.prediction_accuracy) * 100, 1)
         FROM fact_flywheel_engagement fe3
         JOIN dim_date d3 ON fe3.snapshot_date_key = d3.date_key
         WHERE fe3.user_id = ee.user_id
           AND d3.calendar_date BETWEEN ee.expansion_date + INTERVAL '1 day'
                                     AND ee.expansion_date + INTERVAL '7 days'
        ) AS accuracy_after
    FROM expansion_events ee
)
SELECT
    user_id,
    expansion_date,
    accuracy_before,
    accuracy_after,
    accuracy_after - accuracy_before AS accuracy_lift,
    CASE
        WHEN accuracy_after > accuracy_before THEN 'POSITIVE'
        WHEN accuracy_after = accuracy_before THEN 'NEUTRAL'
        ELSE 'NEGATIVE'
    END AS correlation
FROM before_after
WHERE accuracy_before IS NOT NULL
  AND accuracy_after IS NOT NULL
ORDER BY accuracy_lift DESC;
```

### Query 5: Cohort Analysis by Autonomy Stage

Groups users by their current autonomy stage and compares aggregate flywheel metrics. Answers: "How do flywheels at Stage 3 perform compared to Stage 4?"

```sql
SELECT
    pm.autonomy_stage,
    CASE pm.autonomy_stage
        WHEN 1 THEN 'Observe'
        WHEN 2 THEN 'Suggest'
        WHEN 3 THEN 'Predict'
        WHEN 4 THEN 'Act with Approval'
        WHEN 5 THEN 'Autonomous'
    END AS stage_name,
    COUNT(DISTINCT fe.user_id) AS user_count,
    ROUND(AVG(fe.observation_depth), 0) AS avg_observation_depth,
    ROUND(AVG(fe.prediction_accuracy) * 100, 1) AS avg_accuracy_pct,
    ROUND(AVG(fe.action_completion_rate) * 100, 1) AS avg_completion_pct,
    ROUND(AVG(fe.action_reversal_rate) * 100, 1) AS avg_reversal_pct,
    ROUND(AVG(fe.engagement_velocity), 2) AS avg_velocity,
    COUNT(*) FILTER (
        WHERE fe.observation_surface_expansion = TRUE
    ) AS surface_expansions,
    ROUND(AVG(pm.trust_level), 1) AS avg_trust_level
FROM fact_flywheel_engagement fe
JOIN dim_personal_model pm ON fe.model_key = pm.model_key
JOIN dim_date d ON fe.snapshot_date_key = d.date_key
WHERE d.calendar_date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY pm.autonomy_stage
ORDER BY pm.autonomy_stage;
```

### Query 6: Weekly Velocity Trend with Moving Average

Smooths daily velocity fluctuations into a 7-day moving average for executive reporting.

```sql
SELECT
    d.calendar_date,
    ROUND(fe.engagement_velocity, 2) AS daily_velocity,
    ROUND(AVG(fe.engagement_velocity) OVER (
        PARTITION BY fe.user_id
        ORDER BY d.calendar_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ), 2) AS velocity_7d_avg,
    ROUND(AVG(fe.prediction_accuracy) OVER (
        PARTITION BY fe.user_id
        ORDER BY d.calendar_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) * 100, 1) AS accuracy_7d_avg
FROM fact_flywheel_engagement fe
JOIN dim_date d ON fe.snapshot_date_key = d.date_key
WHERE fe.user_id = :user_id
  AND d.calendar_date >= CURRENT_DATE - INTERVAL '90 days'
ORDER BY d.calendar_date;
```

---

## 5.6 ORCHA Analytics Schema Mapping

The `fact_flywheel_engagement` table does not exist in a vacuum. It maps directly to metrics and infrastructure already operating in the ORCHA platform. The following table traces each flywheel KPI to its ORCHA analog.

### KPI-to-ORCHA Mapping

| Flywheel KPI | ORCHA Analog | Source Path | Mapping Logic |
|-------------|-------------|-------------|---------------|
| **Observation Depth** | Moby encoding coverage | `packages/orcha-moby/src/orcha_moby/rag/cli.py` (`status` command) | Total chunks indexed per tenant per day. Higher encoding coverage means more of the knowledge base has been observed and embedded. The `rag status` command reports total documents, chunks, and collections --- the ORCHA equivalent of observation depth. |
| **Prediction Accuracy** | Brain MCP cache hit rate | `mcp-servers/brain/server.py` (query cache layer) | Cache hits indicate that a previous query (prediction) was useful enough to be requested again. The cache hit ratio is a proxy for prediction relevance: queries that are never re-requested were likely not useful. |
| **Action Completion Rate** | Agent task completion | `harness/src/agent_harness/events/` (EventBus events) | The ratio of `task.completed` events to `task.started` events in the EventBus stream. Tasks that start but never complete indicate action layer failures. |
| **Action Reversal Rate** | Git revert / PR close rate | `harness/src/agent_harness/crew/` (crew event stream) | PRs closed without merge, commits reverted, and worktrees abandoned are the ORCHA equivalent of action reversals --- the human operator judged the agent's work to be wrong. |
| **Trust Score Delta** | Autonomy escalation events | `harness/src/agent_harness/dx/trust.py` (`TrustManager`) | Trust score changes when the `TrustManager.verify_or_prompt` pattern either grants or revokes trust for a skill source. Each grant is a positive trust delta; each revocation is negative. |
| **Engagement Velocity** | Composite of above metrics | `packages/orcha-analytics/` (aggregation layer) | No single ORCHA metric maps to velocity. It would be computed by the analytics pipeline from the component metrics above, following the formula in Section 5.3. |
| **Observation Surface Expansion** | New collection creation in Moby | `packages/orcha-moby/src/orcha_moby/rag/storage/` | When a user creates a new Moby collection or ingests documents from a previously unindexed source, the observation surface expands. The `WikiPipeline.ingest()` method for a new `collection_id` is the trigger. |

### Mapping to Existing Fact Tables

| Existing ORCHA Fact Table | Relationship to `fact_flywheel_engagement` | Join Strategy |
|--------------------------|------------------------------------------|---------------|
| `fact_agent_event` | Source data. Agent events are the transactional grain from which daily engagement metrics are aggregated. Event counts feed `observation_depth`; error rates influence `action_completion_rate`. | Aggregate `fact_agent_event` by `user_id` + `date_key` into `fact_flywheel_engagement`. |
| `fact_session_snapshot` | Correlated metric. Session health (`health_score`, `is_healthy`) correlates with flywheel velocity --- unhealthy sessions produce poor engagement. | Join on `user_id` + `snapshot_date_key` for correlation analysis. |
| `fact_task_lifecycle` | Source data for action metrics. Task completion status and duration feed `action_count` and `action_completion_rate`. | Aggregate completed/failed tasks by `user_id` + `date_key`. |
| `fact_resource_snapshot` | Diagnostic context. Resource exhaustion (high CPU, memory alerts) explains velocity drops that are not caused by engagement issues. | Join on `snapshot_date_key` to exclude resource-constrained periods from engagement analysis. |

### IG Freshness as Flywheel Health Indicator

The Inspector General (`packages/orcha-moby/src/orcha_moby/ig/`) monitors knowledge base freshness using four status levels: `SYNCED`, `STALE`, `MISSING`, and `ORPHANED`. These map directly to flywheel health:

| IG Status | Flywheel Interpretation |
|-----------|------------------------|
| `SYNCED` | Observation layer is current. The knowledge graph reflects the latest state of the observed world. Flywheel has fresh fuel. |
| `STALE` | Observations are aging. The inference layer is working from outdated data. Prediction accuracy will decline if staleness persists. |
| `MISSING` | Part of the observation surface has gaps. Documents or sources that should be indexed are not. The flywheel cannot see what it should. |
| `ORPHANED` | The knowledge graph contains entries for sources that no longer exist. Dead weight that may produce hallucinated predictions. |

A healthy flywheel has 90%+ `SYNCED` status across its knowledge base. Dropping below 80% is a leading indicator of velocity decline --- the predictions will start degrading 1--2 days after the knowledge base goes stale.

### Star Schema Position

```
                    ┌──────────────┐
                    │  dim_date    │
                    └──────┬───────┘
                           │
                           │  snapshot_date_key
                           │
┌──────────────────┐       │       ┌────────────────────────┐
│ dim_personal_    │───────┼───────│  fact_flywheel_        │
│ model (SCD2)     │  model_key   │  engagement             │
└──────────────────┘       │       └────────────┬───────────┘
                           │                    │
                           │              user_id (degenerate)
                           │
                    ┌──────┴───────┐
                    │ (tenant_id)  │
                    │  RLS scope   │
                    └──────────────┘
```

The star schema is intentionally narrow. `fact_flywheel_engagement` has only two dimension foreign keys (`dim_date` and `dim_personal_model`) plus one degenerate dimension (`user_id`). This reflects the table's purpose: it measures the *user's flywheel*, not a specific agent, session, or project. Agent-level and session-level detail is available by joining back to `fact_agent_event` and `fact_session_snapshot` through the shared `user_id` and `date_key`.

---

## 5.7 Flywheel Acceleration vs. Stall

The engagement velocity metric and its supporting KPIs define three flywheel states. Detecting which state a user's flywheel is in --- and responding appropriately --- is the primary purpose of `fact_flywheel_engagement`.

### State 1: Accelerating

**Definition:** Engagement velocity is positive and increasing. The flywheel is spinning faster with each revolution.

| Indicator | Threshold | Signal |
|-----------|-----------|--------|
| Engagement velocity | > +5.0 | Composite metric is positive |
| Prediction accuracy | Increasing or stable above 0.70 | Model is learning effectively |
| Action reversal rate | Below 0.05 and stable or declining | User trusts the action layer |
| Trust score delta | Positive (any amount) | Trust is growing |
| Observation surface expansion | TRUE at least once in 30 days | User is actively feeding the flywheel |
| Observation depth | Increasing week-over-week | The flywheel is seeing more |

**Expected behavior:** The flywheel is in its growth phase. Prediction accuracy is climbing because the model has fresh, abundant observations. The user is accepting predictions, completing actions, and expanding the observation surface. Trust builds, which enables higher autonomy stages, which enables more actions, which generates more feedback, which improves predictions. This is the virtuous cycle the entire system is designed to produce.

**System response:** Continue normal operation. Consider surfacing autonomy stage promotion opportunities if the user has been at the current stage for the minimum required duration and all metrics support advancement.

### State 2: Steady

**Definition:** Engagement velocity is near zero (between -5.0 and +5.0). The flywheel is spinning at a constant rate --- healthy but not growing.

| Indicator | Threshold | Signal |
|-----------|-----------|--------|
| Engagement velocity | -5.0 to +5.0 | Near equilibrium |
| Prediction accuracy | Stable between 0.65 and 0.85 | Model has reached its ceiling for current observation surface |
| Action reversal rate | Stable below 0.05 | No trust degradation |
| Trust score delta | Zero or small positive | Trust is flat |
| Observation surface expansion | FALSE for 30+ days | User is content with current scope |
| Observation depth | Stable week-over-week | No growth, no decline |

**Expected behavior:** The flywheel has reached equilibrium for the user's current engagement level. This is a normal and healthy state for mature flywheels --- not every user will continuously expand their observation surface or advance through autonomy stages. A user at Stage 3 (Predict) with stable, high-accuracy predictions is deriving real value.

**System response:** No intervention required. Periodically (e.g., monthly) surface low-friction opportunities to expand the observation surface or demonstrate a new capability that might reignite growth. Do not nag --- a steady flywheel that is pushed too hard will stall.

### State 3: Stalling

**Definition:** Engagement velocity is negative and/or declining. The flywheel is losing momentum.

| Indicator | Threshold | Signal |
|-----------|-----------|--------|
| Engagement velocity | < -5.0 | Composite metric is negative |
| Prediction accuracy | Declining below 0.60 | Model is degrading or environment has changed |
| Action reversal rate | Above 0.10 and/or increasing | User is actively rejecting system actions |
| Trust score delta | Negative | Trust is eroding |
| Observation surface expansion | FALSE, AND observation depth declining | User may be revoking permissions or migrating activity |
| Observation depth | Declining week-over-week | The flywheel is seeing less |

**Root causes of stall:**

| Root Cause | Detection Pattern | Remediation |
|------------|------------------|-------------|
| **Knowledge staleness** | IG reports > 20% STALE/MISSING | Trigger re-sync; notify user of stale data |
| **Environment change** | Observation depth stable but accuracy dropping | The user's patterns changed (new project, new role, new tools). The model needs to re-learn. |
| **Trust breach** | Large negative trust delta following an incident | Automatic autonomy downgrade. System should explicitly acknowledge the failure and reduce its own privileges. |
| **Over-autonomy** | High reversal rate at Stage 4/5 | Demote to previous autonomy stage. The system escalated faster than the model's accuracy supports. |
| **Platform migration** | Observation depth declining while accuracy is stable | User is moving activity to a different device or platform. The flywheel is being starved. |
| **Notification fatigue** | Prediction count high but acceptance rate low | Reduce prediction frequency. Quality over quantity. |

### Detection Queries

**Stall Alert: Users with 3+ consecutive days of negative velocity.**

```sql
-- Reuses the stall detection query from Section 5.5, Query 3.
-- Configure as a scheduled job (e.g., dbt snapshot, pg_cron, or
-- ORCHA analytics pipeline) running daily at 01:00 UTC.

-- Output: List of users with stall_days >= 3, ordered by severity.
-- Action: Trigger alert to platform operations team.
```

**Trust Crisis Alert: Users with trust delta < -5 in a single day.**

```sql
SELECT
    fe.user_id,
    d.calendar_date,
    fe.trust_score_delta,
    fe.engagement_velocity,
    ROUND(fe.action_reversal_rate * 100, 1) AS reversal_pct,
    pm.trust_level AS current_trust,
    pm.autonomy_stage AS current_stage
FROM fact_flywheel_engagement fe
JOIN dim_date d ON fe.snapshot_date_key = d.date_key
JOIN dim_personal_model pm
    ON pm.user_id = fe.user_id AND pm.is_current = TRUE
WHERE fe.trust_score_delta < -5
  AND d.calendar_date = CURRENT_DATE - INTERVAL '1 day'
ORDER BY fe.trust_score_delta ASC;
```

**Automatic Autonomy Downgrade: Users with reversal rate above threshold.**

```sql
-- Identify users who should be automatically demoted one autonomy stage.
-- Criteria: reversal rate > 10% for 5+ of the last 7 days.

WITH reversal_check AS (
    SELECT
        fe.user_id,
        COUNT(*) FILTER (
            WHERE fe.action_reversal_rate > 0.10
        ) AS high_reversal_days,
        ROUND(AVG(fe.action_reversal_rate) * 100, 1) AS avg_reversal_pct
    FROM fact_flywheel_engagement fe
    JOIN dim_date d ON fe.snapshot_date_key = d.date_key
    WHERE d.calendar_date >= CURRENT_DATE - INTERVAL '7 days'
      AND fe.action_count > 0  -- only consider days with actions
    GROUP BY fe.user_id
    HAVING COUNT(*) FILTER (WHERE fe.action_reversal_rate > 0.10) >= 5
)
SELECT
    rc.user_id,
    rc.high_reversal_days,
    rc.avg_reversal_pct,
    pm.autonomy_stage AS current_stage,
    pm.autonomy_stage - 1 AS recommended_stage,
    pm.trust_level
FROM reversal_check rc
JOIN dim_personal_model pm
    ON pm.user_id = rc.user_id AND pm.is_current = TRUE
WHERE pm.autonomy_stage > 1;  -- Cannot demote below Stage 1
```

### Alert Thresholds Summary

| Alert Level | Condition | Response |
|-------------|-----------|----------|
| **INFO** | Velocity between -5 and 0 for 1 day | Log for trend analysis. No action. |
| **WARNING** | Velocity < -5 for 2 consecutive days | Flag in daily operations dashboard. |
| **HIGH** | Velocity < -5 for 3+ consecutive days, OR reversal rate > 10% | Notify platform team. Investigate root cause. |
| **CRITICAL** | Trust delta < -5 in single day, OR reversal rate > 20% | Automatic autonomy downgrade. Notify user with explanation. Log incident for review. |

These alert levels mirror ORCHA's existing `AlertLevel` hierarchy in the IG Monitor (`agent_harness.ig.monitor`), maintaining consistency across the platform's observability stack.

---

## Summary

`fact_flywheel_engagement` is the periodic snapshot fact table that transforms the flywheel from a conceptual model into a measurable system. By capturing daily snapshots of observation depth, prediction accuracy, action completion, trust movement, and composite velocity, it provides the instrumentation needed to answer the question that matters most: is this flywheel spinning faster, holding steady, or stalling?

Key design decisions:

- **Periodic snapshot (daily), not transactional.** Flywheel health is measured by rates of change across days, not individual events. Pre-aggregating into daily snapshots enables efficient dashboard queries and trend analysis.
- **One row per user per day grain.** The flywheel belongs to the user, not to an agent, session, or project. The grain reflects this: each row measures one person's flywheel on one day.
- **Engagement velocity as a composite metric.** No single KPI captures flywheel health. Velocity combines prediction accuracy (weight 40), action completion (weight 25), action reversal (weight -50, intentionally heavy), trust delta (weight 10), and surface expansion (weight 15) into a single diagnostic scalar.
- **Asymmetric trust weighting.** Reversal rate carries double the negative weight of completion's positive weight, reflecting the empirical reality that trust destruction is faster than trust construction. One wrong action undoes many correct ones.
- **Three flywheel states with automated response.** Accelerating (velocity > +5), Steady (-5 to +5), and Stalling (< -5) are not just labels --- they trigger specific system behaviors, from autonomy promotion opportunities to automatic autonomy downgrades.
- **Direct ORCHA mapping.** Every KPI maps to an existing ORCHA component: Moby encoding coverage for observation depth, Brain MCP cache hits for prediction accuracy, EventBus task events for action completion, and TrustManager grants for trust deltas. The flywheel OS model is not theoretical --- it describes metrics ORCHA already collects.
- **IG freshness as a leading indicator.** Knowledge base staleness (Moby IG status) predicts engagement velocity drops 1--2 days before they appear in the engagement metrics, providing an early warning system for flywheel stall.

The next chapter examines the trust architecture that determines the flywheel's maximum velocity --- because a flywheel that users do not trust will never accelerate past Stage 1, regardless of how good the underlying model becomes.

---

*Next: [Chapter 6 --- The Trust Problem: Why the Flywheel Can Stall](ch06-trust-architecture.md)*
