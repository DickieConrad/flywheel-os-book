# The Personal Data Flywheel OS

**An AI-Embedded Operating System Architecture**

*Strategic Plan & Technical Blueprint*

> From program loader to self-defending networked system to personal intelligence engine.
> The next SCD2 inflection point is the OS that knows you --- and gets smarter the more you use it.

---

## Table of Contents

| # | Chapter | Description | Key Artifacts |
|---|---------|-------------|---------------|
| 1 | [The Historical Arc](ch01-historical-arc.md) | Why the AI-embedded OS is inevitable. OS evolution mapped to SCD2 inflection points. | `dim_os_era` dimension, era timeline |
| 2 | [The Personal Data Flywheel](ch02-flywheel-mechanics.md) | The five-stage flywheel loop: Observe, Infer, Predict, Act, Learn. Current precedents. | Flywheel cycle diagram, stage definitions |
| 3 | [Technical Architecture](ch03-technical-architecture.md) | The seven-layer Flywheel OS stack with ORCHA equivalents for every layer. | Architecture diagram, component matrix, hardware requirements |
| 4 | [The New Kimball Dimension](ch04-kimball-dimension.md) | `dim_personal_model` SCD Type 2 dimension table design. | PostgreSQL DDL, SCD2 triggers, GDPR mapping |
| 5 | [Flywheel Health Metrics](ch05-health-metrics.md) | `fact_flywheel_engagement` periodic snapshot fact table. Six KPIs with composite velocity. | DDL, analytics queries, stall detection |
| 6 | [The Trust Problem](ch06-trust-architecture.md) | Why the flywheel can stall. Failure modes and trust architecture principles. | Trust scoring rubric, case studies |
| 7 | [Implementation Roadmap](ch07-implementation-roadmap.md) | Five-phase plan mapped to flywheel stages, with ORCHA validation evidence. | Gantt chart, dependency graph, risk register |
| 8 | [Strategic Implications](ch08-strategic-implications.md) | OS-as-moat, app model inversion, privacy-as-feature, compliance explosion. | Competitive matrix, three-tenant analysis |
| A | [Appendix: RLM Integration](appendix-a-rlm-integration.md) | Recursive Language Models for unbounded context in the Flywheel OS. | RLM-to-ORCHA mapping, integration spec |

---

## The Seven-Layer Stack

```
Layer 7: App Context APIs          "What can apps see?"
Layer 6: Privacy Gateway            "What can leave the device?"
Layer 5: Action Layer               "What can the system do?"
Layer 4: Personal Knowledge Graph   "What does the system know?"
Layer 3: Foundation Model           "How does the system think?"
Layer 2: Secure Enclave             "Where is trust rooted?"
Layer 1: NPU Layer                  "What runs the inference?"
```

Each layer maps to an ORCHA component. See [Chapter 3](ch03-technical-architecture.md) for the full mapping table.

---

## Kimball Models

| Model | Type | Chapter | Purpose |
|-------|------|---------|---------|
| `dim_personal_model` | SCD Type 2 Dimension | [Ch. 4](ch04-kimball-dimension.md) | Track personal model state over time |
| `fact_flywheel_engagement` | Periodic Snapshot Fact | [Ch. 5](ch05-health-metrics.md) | Measure flywheel acceleration/stall |

---

## ORCHA Layer Mapping

| # | Flywheel OS Layer | ORCHA Component | Key Source |
|---|------------------|----------------|------------|
| 1 | NPU Layer | Brain MCP backend selection | `mcp-servers/brain/server.py` |
| 2 | Secure Enclave | Supabase tenant JWT + RLS | `packages/orcha-moby/src/orcha_moby/storage/supabase_auth.py` |
| 3 | Foundation Model | Brain MCP Server | `mcp-servers/brain/` |
| 4 | Knowledge Graph | Moby RAG | `packages/orcha-moby/` |
| 5 | Action Layer | Agent Harness + Event Bus | `harness/src/agent_harness/` |
| 6 | Privacy Gateway | Tenant isolation + config | `packages/orcha-moby/src/orcha_moby/storage/` |
| 7 | App Context APIs | MCP Server tools + Analytics | `mcp-servers/`, `packages/orcha-analytics/` |

---

## Glossary

| Term | Definition |
|------|-----------|
| **Flywheel** | Self-reinforcing loop where OS utility increases with personal data depth, increasing usage, generating more data |
| **SCD Type 2** | Slowly Changing Dimension pattern that preserves full history by creating new rows on change |
| **NPU** | Neural Processing Unit --- dedicated inference hardware (40+ TOPS) |
| **Secure Enclave** | Hardware-isolated trust root for personal data (Apple SEP, Microsoft Pluton, ARM TrustZone) |
| **Engagement Velocity** | Composite metric measuring flywheel health: weighted sum of prediction accuracy, action completion, reversal rate, trust delta |
| **Graduated Autonomy** | Five-stage trust progression: Observe, Suggest, Predict, Act with Approval, Autonomous |
| **Observation Surface** | The set of data sources the OS is permitted to observe (email, calendar, files, etc.) |
| **Privacy Gateway** | Layer 6 --- all cloud requests pass through differential privacy and audit logging |
| **LoRA** | Low-Rank Adaptation --- per-user model personalization without full retraining |
| **Cryptographic Erasure** | GDPR-compliant deletion by destroying encryption keys rather than deleting rows |

---

## Reading Order

**For architects:** Ch. 3 (Architecture) → Ch. 4 (Dimension) → Ch. 5 (Metrics) → Ch. 7 (Roadmap)

**For strategists:** Ch. 1 (History) → Ch. 2 (Flywheel) → Ch. 6 (Trust) → Ch. 8 (Strategy)

**For implementers:** Ch. 3 (Architecture) → Ch. 7 (Roadmap) → Ch. 4 (Dimension) → Ch. 5 (Metrics) → Appendix A (RLM)

---

## Statistics

- **Chapters:** 8 + 1 Appendix
- **Total lines:** 4,723
- **Kimball models:** 2 (1 dimension, 1 fact)
- **PostgreSQL DDL:** 2 complete table definitions with indexes, RLS, constraints
- **Analytics queries:** 11+
- **ORCHA component mappings:** 7 layers fully mapped
- **Competitive analyses:** 5 platforms (Apple, Microsoft, Google, Meta, Open/Linux)
- **Risk register items:** 5 with severity ratings and mitigations

---

*Source: [The Personal Data Flywheel OS --- Strategic Plan](../../references/personal-data-flywheel-os.md)*

*Built with ORCHA meta-ralph parallel orchestration. PRs #307--#316.*
