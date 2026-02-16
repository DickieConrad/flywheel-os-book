# Chapter 4 — The New Kimball Dimension: `dim_personal_model`

> *"The personal model IS the flywheel's state—and it changes with every interaction."*

Every era of operating system history introduced slowly-changing attributes that didn't exist before. The standalone OS tracked hardware configuration. The networked OS added TCP/IP state, DNS resolution, firewall rules. The security OS added patch levels, certificate stores, compliance posture. Each transition added new columns to the OS dimension table and new triggers that changed the meaning of "current state."

The AI-embedded OS introduces the most consequential dimension yet: the personal model. This chapter designs the Kimball-style slowly-changing dimension table that tracks it.

---

## 4.1 Why SCD Type 2?

Ralph Kimball's Slowly Changing Dimension (SCD) taxonomy classifies how dimension attributes change over time:

| SCD Type | Behavior | Use Case |
|----------|----------|----------|
| Type 1 | Overwrite old value | Corrections (typo in a name) |
| Type 2 | Add new row, expire old row | Track history (what was true *then*?) |
| Type 3 | Add column for previous value | Track limited history (current + prior) |
| Type 6 (Hybrid) | Combine Types 1, 2, 3 | Current value + history + previous |

The personal model demands **Type 2** — full historical tracking — for three reasons:

1. **Temporal queries are essential.** "What did the OS know about me on March 15?" is a first-class question for debugging, compliance, and trust. Type 1 (overwrite) destroys this capability.

2. **Regulatory compliance requires auditability.** GDPR Article 15 (right of access) and Article 17 (right to erasure) require the system to answer "what data did you hold about me at time T?" and "prove you deleted it." SCD2 provides the audit trail.

3. **The flywheel compounds over time.** The value of the personal model is precisely that it remembers. Overwriting previous state would break the flywheel's core mechanic — the compounding of learned context.

### SCD2 Mechanics

When an SCD2 attribute changes:

1. The **current row** gets its `effective_to` timestamp set to `now()` and `is_current` set to `FALSE`.
2. A **new row** is inserted with `effective_from = now()`, `effective_to = '9999-12-31'`, and `is_current = TRUE`.
3. The **surrogate key** (`model_key`) auto-increments. The **natural key** (`user_id`) stays the same across rows.

This means a single user may have hundreds or thousands of rows in `dim_personal_model` — one for every version of their personal model. The `is_current = TRUE` filter always returns exactly one row per user.

---

## 4.2 The `dim_personal_model` Table

### Column Reference

| Column | Type | Description |
|--------|------|-------------|
| `model_key` | `BIGSERIAL` | Surrogate key (auto-increment). Primary key. |
| `user_id` | `UUID` | Natural key. Stable across all SCD2 versions. |
| `model_version` | `INTEGER` | Monotonically increasing version counter per user. |
| `embedding_hash` | `TEXT` | SHA-256 hash of the user's current embedding vector. Detects when the semantic representation changed without storing the raw vector. |
| `preference_vector_hash` | `TEXT` | SHA-256 hash of the user's preference vector. Captures learned preferences (topics, tools, workflows, communication style). |
| `knowledge_graph_snapshot_id` | `UUID` | Foreign key to a stored snapshot of the user's personal knowledge graph. The graph contains entities, relationships, and temporal patterns. |
| `trust_level` | `SMALLINT` | 0–100 trust score. Starts at 0 for new users. Increases as the system makes accurate predictions and the user expands the observation surface. |
| `autonomy_stage` | `SMALLINT` | 1–5 graduated autonomy stage (see below). |
| `observation_surface_scope` | `JSONB` | Describes what the OS is permitted to observe. Example: `{"email": true, "calendar": true, "browser": false, "files": ["~/Documents"]}` |
| `effective_from` | `TIMESTAMPTZ` | When this version became current. |
| `effective_to` | `TIMESTAMPTZ` | When this version was superseded. `'9999-12-31'` for current row. |
| `is_current` | `BOOLEAN` | Convenience flag. Always `TRUE` for the latest version. |
| `tenant_id` | `TEXT` | Tenant scope for RLS (ORCHA multi-tenant pattern). |
| `created_at` | `TIMESTAMPTZ` | Row insertion timestamp. |

### Autonomy Stages

The `autonomy_stage` column tracks the graduated trust relationship between user and OS:

| Stage | Name | Behavior |
|-------|------|----------|
| 1 | **Observe** | OS watches but takes no action. Builds the knowledge graph silently. |
| 2 | **Suggest** | OS offers contextual suggestions (e.g., "You usually schedule 1:1s on Tuesdays"). User must explicitly accept. |
| 3 | **Predict** | OS pre-computes likely actions (e.g., pre-drafts meeting agenda). Higher confidence threshold required. |
| 4 | **Act with Approval** | OS executes actions but requires explicit approval (e.g., "Send this email? [Yes/No]"). |
| 5 | **Autonomous** | OS acts independently within approved scopes. User is notified post-hoc. Full undo available. |

**The system never skips stages.** A user at Stage 2 cannot be promoted to Stage 4 regardless of trust score. Each stage must be earned through demonstrated accuracy at the current level.

### Grain

The grain of `dim_personal_model` is **one row per version of one user's personal model**. The natural key is `user_id`. The surrogate key is `model_key`. Every time any SCD2-tracked attribute changes, a new row is created.

---

## 4.3 SCD2 Trigger Logic

### What Causes a New Row?

Not every interaction creates a new version. Only **material state changes** trigger SCD2 row creation:

| Trigger | Columns Affected | Frequency |
|---------|------------------|-----------|
| Embedding re-computation | `embedding_hash` | Daily (batch) or on significant new data |
| Preference learning update | `preference_vector_hash` | Weekly or on explicit preference change |
| Knowledge graph snapshot | `knowledge_graph_snapshot_id` | Daily (batch) or on major entity addition |
| Trust score threshold crossed | `trust_level` | On prediction accuracy milestone |
| Autonomy stage promotion | `autonomy_stage` | Rare — requires sustained trust at current stage |
| Observation surface change | `observation_surface_scope` | On user permission grant/revoke |

### Why Not Every Interaction?

If the system created a new SCD2 row for every micro-update (e.g., every email read, every file opened), the dimension would explode. Instead, the personal model uses a **batch materialization** pattern:

1. **Continuous ingestion** writes raw events to a staging area (analogous to ORCHA's EventBus JSONL logs).
2. **Periodic materialization** (e.g., nightly) recomputes the embedding, preference vector, and knowledge graph snapshot.
3. **Hash comparison** determines whether the materialized state differs from the current SCD2 row.
4. **Only on change** does the SCD2 trigger fire.

This keeps the dimension manageable — typically 1–3 new rows per user per week during active use, with long stretches of stability for inactive users.

### Trigger Pseudocode

```sql
-- SCD2 trigger: expire current row and insert new version
CREATE OR REPLACE FUNCTION scd2_personal_model_trigger()
RETURNS TRIGGER AS $$
BEGIN
    -- Only fire if a tracked attribute actually changed
    IF (OLD.embedding_hash IS DISTINCT FROM NEW.embedding_hash
        OR OLD.preference_vector_hash IS DISTINCT FROM NEW.preference_vector_hash
        OR OLD.knowledge_graph_snapshot_id IS DISTINCT FROM NEW.knowledge_graph_snapshot_id
        OR OLD.trust_level IS DISTINCT FROM NEW.trust_level
        OR OLD.autonomy_stage IS DISTINCT FROM NEW.autonomy_stage
        OR OLD.observation_surface_scope IS DISTINCT FROM NEW.observation_surface_scope)
    THEN
        -- Expire the current row
        UPDATE dim_personal_model
        SET effective_to = now(),
            is_current = FALSE
        WHERE user_id = OLD.user_id
          AND is_current = TRUE;

        -- Insert new version
        INSERT INTO dim_personal_model (
            user_id, model_version,
            embedding_hash, preference_vector_hash,
            knowledge_graph_snapshot_id, trust_level,
            autonomy_stage, observation_surface_scope,
            effective_from, effective_to, is_current,
            tenant_id
        ) VALUES (
            NEW.user_id, OLD.model_version + 1,
            NEW.embedding_hash, NEW.preference_vector_hash,
            NEW.knowledge_graph_snapshot_id, NEW.trust_level,
            NEW.autonomy_stage, NEW.observation_surface_scope,
            now(), '9999-12-31'::timestamptz, TRUE,
            NEW.tenant_id
        );

        RETURN NULL; -- Suppress the original UPDATE
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

> **Note:** In practice, the SCD2 logic is typically handled by the application layer (e.g., ORCHA's IngestionPipeline) rather than a database trigger, because the materialization step (recomputing embeddings and hashes) happens in Python. The trigger above illustrates the logical pattern.

---

## 4.4 PostgreSQL DDL

The following DDL is compatible with Supabase (PostgreSQL 15+) and follows ORCHA's existing migration patterns.

```sql
-- =============================================================================
-- dim_personal_model: SCD Type 2 Dimension for Personal Model State
-- =============================================================================
-- Tracks the evolving state of a user's personal model over time.
-- Each row represents one version of one user's model.
-- The is_current flag and effective_from/effective_to brackets enable
-- both point-in-time queries and current-state lookups.
--
-- Grain: One row per version per user.
-- Natural key: user_id
-- Surrogate key: model_key
-- =============================================================================

CREATE TABLE IF NOT EXISTS dim_personal_model (
    -- Surrogate key (auto-increment)
    model_key       BIGSERIAL PRIMARY KEY,

    -- Natural key (stable across versions)
    user_id         UUID NOT NULL,

    -- Version tracking
    model_version   INTEGER NOT NULL DEFAULT 1,

    -- Model state hashes (detect change without storing raw vectors)
    embedding_hash          TEXT,
    preference_vector_hash  TEXT,

    -- Knowledge graph reference
    knowledge_graph_snapshot_id  UUID,

    -- Trust and autonomy
    trust_level     SMALLINT NOT NULL DEFAULT 0
                    CHECK (trust_level BETWEEN 0 AND 100),
    autonomy_stage  SMALLINT NOT NULL DEFAULT 1
                    CHECK (autonomy_stage BETWEEN 1 AND 5),

    -- Observation permissions
    observation_surface_scope  JSONB NOT NULL DEFAULT '{}'::jsonb,

    -- SCD Type 2 tracking columns
    effective_from  TIMESTAMPTZ NOT NULL DEFAULT now(),
    effective_to    TIMESTAMPTZ NOT NULL DEFAULT '9999-12-31T23:59:59Z'::timestamptz,
    is_current      BOOLEAN NOT NULL DEFAULT TRUE,

    -- Multi-tenant isolation (ORCHA pattern)
    tenant_id       TEXT NOT NULL,

    -- Metadata
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Constraints
    CONSTRAINT uq_user_current UNIQUE (user_id, is_current)
        WHERE (is_current = TRUE),
    CONSTRAINT chk_effective_range
        CHECK (effective_to >= effective_from),
    CONSTRAINT chk_autonomy_progression
        CHECK (autonomy_stage >= 1 AND autonomy_stage <= 5)
);

-- =============================================================================
-- Indexes
-- =============================================================================

-- Primary access pattern: current state for a user
CREATE INDEX idx_personal_model_current
    ON dim_personal_model (user_id)
    WHERE is_current = TRUE;

-- Point-in-time queries: "what did the OS know on date X?"
CREATE INDEX idx_personal_model_temporal
    ON dim_personal_model (user_id, effective_from, effective_to);

-- Tenant isolation
CREATE INDEX idx_personal_model_tenant
    ON dim_personal_model (tenant_id)
    WHERE is_current = TRUE;

-- Trust level analysis (e.g., "how many users are at autonomy stage 4+?")
CREATE INDEX idx_personal_model_autonomy
    ON dim_personal_model (autonomy_stage, trust_level)
    WHERE is_current = TRUE;

-- =============================================================================
-- Row-Level Security (Supabase pattern)
-- =============================================================================

ALTER TABLE dim_personal_model ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON dim_personal_model
    USING (tenant_id = current_setting('app.tenant_id', TRUE));

-- =============================================================================
-- Comments
-- =============================================================================

COMMENT ON TABLE dim_personal_model IS
    'SCD Type 2 dimension tracking personal model state over time. '
    'Each row represents one version of one user''s personal model. '
    'Use is_current = TRUE for latest state, effective_from/to for point-in-time.';

COMMENT ON COLUMN dim_personal_model.model_key IS
    'Surrogate key. Auto-incrementing. Used as FK from fact tables.';
COMMENT ON COLUMN dim_personal_model.user_id IS
    'Natural key. Stable UUID identifying the user across all SCD2 versions.';
COMMENT ON COLUMN dim_personal_model.embedding_hash IS
    'SHA-256 of the user embedding vector. Change detection without storing raw vector.';
COMMENT ON COLUMN dim_personal_model.trust_level IS
    '0-100 trust score. Increases with prediction accuracy and user engagement.';
COMMENT ON COLUMN dim_personal_model.autonomy_stage IS
    '1=Observe, 2=Suggest, 3=Predict, 4=Act with Approval, 5=Autonomous.';
COMMENT ON COLUMN dim_personal_model.observation_surface_scope IS
    'JSONB describing what the OS is permitted to observe (email, calendar, files, etc).';
```

The full DDL is also available as a migration file at [`supabase/migrations/dim_personal_model.sql`](../../../supabase/migrations/dim_personal_model.sql).

---

## 4.5 Query Examples

### Current State: "What does the OS know about me right now?"

```sql
SELECT
    user_id,
    model_version,
    trust_level,
    autonomy_stage,
    observation_surface_scope,
    effective_from AS current_since
FROM dim_personal_model
WHERE user_id = :user_id
  AND is_current = TRUE;
```

### Point-in-Time: "What did the OS know about me on March 15?"

This is the SCD2 canonical query — finding the row that was "current" at a specific historical timestamp.

```sql
SELECT
    user_id,
    model_version,
    embedding_hash,
    preference_vector_hash,
    trust_level,
    autonomy_stage,
    observation_surface_scope,
    effective_from,
    effective_to
FROM dim_personal_model
WHERE user_id = :user_id
  AND effective_from <= '2026-03-15T00:00:00Z'
  AND effective_to   >  '2026-03-15T00:00:00Z';
```

### Model Evolution: "How has my trust level changed over time?"

```sql
SELECT
    model_version,
    trust_level,
    autonomy_stage,
    effective_from,
    effective_to,
    effective_to - effective_from AS duration
FROM dim_personal_model
WHERE user_id = :user_id
ORDER BY model_version ASC;
```

### Autonomy Stage Distribution: "How many users are at each stage?"

```sql
SELECT
    autonomy_stage,
    CASE autonomy_stage
        WHEN 1 THEN 'Observe'
        WHEN 2 THEN 'Suggest'
        WHEN 3 THEN 'Predict'
        WHEN 4 THEN 'Act with Approval'
        WHEN 5 THEN 'Autonomous'
    END AS stage_name,
    count(*) AS user_count,
    avg(trust_level) AS avg_trust
FROM dim_personal_model
WHERE is_current = TRUE
GROUP BY autonomy_stage
ORDER BY autonomy_stage;
```

### Observation Surface Expansion: "Which users granted email access this month?"

```sql
WITH versioned AS (
    SELECT
        user_id,
        model_version,
        observation_surface_scope->>'email' AS email_access,
        effective_from,
        LAG(observation_surface_scope->>'email')
            OVER (PARTITION BY user_id ORDER BY model_version) AS prev_email_access
    FROM dim_personal_model
    WHERE user_id = :user_id
)
SELECT *
FROM versioned
WHERE email_access = 'true'
  AND (prev_email_access IS NULL OR prev_email_access = 'false')
  AND effective_from >= date_trunc('month', now());
```

### Model Churn: "How frequently do personal models change?"

```sql
SELECT
    user_id,
    count(*) AS total_versions,
    min(effective_from) AS first_version,
    max(effective_from) AS latest_version,
    count(*) FILTER (
        WHERE effective_from >= now() - INTERVAL '30 days'
    ) AS versions_last_30d
FROM dim_personal_model
GROUP BY user_id
ORDER BY versions_last_30d DESC
LIMIT 20;
```

---

## 4.6 ORCHA Analytics Schema Mapping

The `dim_personal_model` dimension integrates with ORCHA's existing Kimball analytics constellation. The existing schema follows patterns established in `packages/orcha-analytics/src/orcha_analytics/models/dimensions.py` and the workbook at `workbooks/kimball-dimensional-modeling/`.

### Conformed Dimension Integration

| Existing ORCHA Dimension | Relationship to `dim_personal_model` |
|--------------------------|--------------------------------------|
| `DimAgent` | Agents operate *on behalf of* a personal model. The agent's `capability_scope` maps to the model's `observation_surface_scope`. |
| `DimSession` | Sessions are *interactions with* a personal model. Each session potentially triggers model re-materialization. |
| `DimDate` / `DimTime` | The `effective_from` and `effective_to` columns in `dim_personal_model` join to `DimDate` for temporal analysis. |
| `DimProject` | Projects provide context for model updates — working on a new project expands the knowledge graph. |

### Fact Table Relationships

| Fact Table | Join Key | Analysis Enabled |
|------------|----------|------------------|
| `fact_agent_event` | `dim_personal_model.model_key` via event timestamp bracket | "Which model version was active when this agent acted?" |
| `fact_session_snapshot` | `dim_personal_model.user_id` + temporal join | "How does session health correlate with model maturity?" |
| `fact_task_lifecycle` | `dim_personal_model.model_key` via accumulating snapshot | "Do tasks complete faster as the model matures?" |
| `fact_flywheel_engagement` (Ch. 5) | `dim_personal_model.model_key` | "Is the flywheel accelerating for this model version?" |

### SCD2 Pattern Consistency

ORCHA's existing `DimensionMember` model (in `packages/orcha-moby/src/orcha_moby/olap/models.py`) already supports SCD2 semantics with `is_current`, `effective_date`, and `expiration_date` fields. The `dim_personal_model` table follows the same pattern but with domain-specific columns. The Pydantic model validator `_validate_scd2_dates` ensures date consistency — the same validation logic applies here: `expiration_date >= effective_date`, and expired members must have an `expiration_date`.

### Star Schema: Personal Model Analytics

```
                    ┌──────────────┐
                    │  DimDate     │
                    └──────┬───────┘
                           │
┌──────────────────┐       │       ┌────────────────────┐
│ DimAgent         │───────┼───────│ fact_flywheel_     │
└──────────────────┘       │       │ engagement         │
                           │       └────────┬───────────┘
                    ┌──────┴───────┐        │
                    │dim_personal_ │────────┘
                    │model         │
                    └──────────────┘
```

---

## 4.7 Data Retention and GDPR Compliance

The personal model dimension stores the most sensitive data in the flywheel OS. Compliance isn't an afterthought — it's a structural requirement.

### GDPR Requirements Mapping

| GDPR Article | Requirement | `dim_personal_model` Implementation |
|-------------|-------------|-------------------------------------|
| Art. 13/14 | Right to be informed | `observation_surface_scope` column documents exactly what is observed. The `model_version` trail shows what was learned and when. |
| Art. 15 | Right of access | Point-in-time query (Section 4.5) answers "what did you know about me on date X?" |
| Art. 16 | Right to rectification | SCD2 naturally supports correction — create a new row with corrected data. The old (incorrect) row remains in history with appropriate annotation. |
| Art. 17 | Right to erasure ("right to be forgotten") | See Cryptographic Erasure below. |
| Art. 20 | Right to data portability | The `dim_personal_model` table can be exported as JSON/CSV per user. Combined with knowledge graph snapshot, this provides a complete portable identity. |
| Art. 25 | Data protection by design | RLS enforces tenant isolation. Embedding hashes (not raw vectors) are stored. Observation scope requires explicit user grant. |

### Cryptographic Erasure

Traditional `DELETE FROM dim_personal_model WHERE user_id = :id` removes rows but doesn't guarantee erasure from backups, replicas, or WAL logs. The flywheel OS uses **cryptographic erasure**:

1. Each user's personal model data is encrypted with a user-specific key derived from their biometric + device binding.
2. "Deletion" means destroying the encryption key — not deleting rows.
3. The rows remain in the database but are unreadable without the key.
4. This satisfies GDPR's "erasure" requirement because the data is cryptographically inaccessible.

```sql
-- Soft-delete: mark all versions as erased
UPDATE dim_personal_model
SET embedding_hash = '[ERASED]',
    preference_vector_hash = '[ERASED]',
    knowledge_graph_snapshot_id = NULL,
    observation_surface_scope = '{}'::jsonb,
    trust_level = 0,
    autonomy_stage = 1
WHERE user_id = :user_id;

-- The encryption key for this user is separately destroyed
-- in the Secure Enclave key management system.
```

### Retention Policy

| Data Category | Retention Period | Rationale |
|--------------|-----------------|-----------|
| Current model (is_current = TRUE) | Indefinite (while account active) | Required for system operation |
| Historical versions (is_current = FALSE) | 2 years | Regulatory audit window (GDPR allows reasonable retention for compliance) |
| Erased models (post-deletion) | 30 days in backup, then purged | Backup rotation cycle |
| Knowledge graph snapshots | 1 year | Storage cost vs. audit value tradeoff |

### Retention Enforcement

```sql
-- Automated retention: purge expired historical versions older than 2 years
DELETE FROM dim_personal_model
WHERE is_current = FALSE
  AND effective_to < now() - INTERVAL '2 years';
```

### Audit Trail

Every SCD2 row transition is itself an audit event. The `effective_from` and `effective_to` timestamps, combined with the `model_version` sequence, provide a complete chain of custody for the personal model's evolution. This trail answers the compliance question: **"When did you learn this about me, and when did you stop knowing it?"**

---

## Summary

The `dim_personal_model` table is the Kimball dimension that makes the flywheel OS auditable, queryable, and compliant. It tracks the most important slowly-changing state in the system — the user's personal model — using the same SCD2 patterns that ORCHA already uses for agent and session dimensions.

Key design decisions:

- **SCD Type 2** for full historical tracking (not Type 1 overwrite)
- **Hash-based change detection** to avoid storing raw embeddings in the dimension
- **Batch materialization** to keep row count manageable (1–3 per user per week, not per interaction)
- **Graduated autonomy stages** that can never be skipped
- **Cryptographic erasure** for GDPR-compliant deletion
- **Tenant-scoped RLS** following ORCHA's existing Supabase patterns

The next chapter designs `fact_flywheel_engagement` — the periodic snapshot fact table that measures whether this flywheel is accelerating or stalling.
