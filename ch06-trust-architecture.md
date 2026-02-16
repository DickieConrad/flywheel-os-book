# Chapter 6: The Trust Problem: Why the Flywheel Can Stall

> *"Security is a process, not a product."* -- Bruce Schneier
>
> In the AI flywheel era, trust is the process. And most companies have shipped the product without it.

---

Every historical OS transition faced resistance rooted in a single question: *Should I let this thing do that?* TCP/IP meant exposure to network worms. Automatic updates meant ceding control of your own machine. Cloud sync meant your files living on someone else's server. Each time, the OS asked for more trust. Each time, the early implementations got it wrong. Each time, the companies that solved trust won the era.

The AI flywheel's ask is the largest yet: *Let me observe everything you do, learn your patterns, and act on your behalf.* The early evidence---from Microsoft, Apple, and Google---suggests the industry is repeating history's mistakes.

This chapter examines three case studies of trust failure, derives five architectural principles for getting trust right, and maps those principles to ORCHA's existing security model as a reference implementation.

---

## 6.1 Three Case Studies of Trust Failure

### Case Study 1: Microsoft Recall --- The Plaintext Debacle

**What happened.** In May 2024, Microsoft announced Recall as a headline feature of Copilot+ PCs. The premise was compelling: your computer would take screenshots every few seconds, OCR the content, and let you search your entire computing history semantically. A personal time machine.

**Where trust broke.** Security researcher Kevin Beaumont discovered that Recall stored all screenshot data---every password typed, every private message displayed, every medical record viewed---in an unencrypted SQLite database in the user's AppData folder. Any application running under the user's account could read it. Any malware with user-level access had a complete record of everything the user had ever done on the machine.

Within days, Alex Hagenah released "TotalRecall," a proof-of-concept tool that extracted and displayed the entire Recall database in seconds. The security community's response was unequivocal: Signal, Brave, and AdGuard all shipped features specifically designed to block Recall's screenshot capture.

**The fallout.** Microsoft delayed the feature by a full year. When it relaunched in 2025, Recall was opt-in (not opt-out), required Windows Hello biometric authentication, encrypted the database at rest, and filtered sensitive content. By early 2026, Microsoft internally considered the feature to have "failed"---not because the concept was wrong, but because the initial implementation destroyed the trust required for adoption.

**The lesson.** *Security-by-policy is not security.* Microsoft's original Recall relied on the assumption that user-level access control was sufficient protection for the most sensitive data a computer could collect. That assumption was wrong for the network era (where firewalls replaced file permissions as the trust boundary), and it was catastrophically wrong for the AI era. The flywheel demands *hardware-enforced* security boundaries, not software promises.

| Dimension | Score (1-5) | Assessment |
|-----------|:-----------:|------------|
| Data Protection | 1 | Plaintext SQLite, no encryption at rest |
| User Control | 2 | Opt-out by default, no granular exclusions |
| Transparency | 2 | Users couldn't see what was being captured in real-time |
| Reversibility | 3 | Data could be deleted, but no cryptographic guarantee |
| Value Return | 4 | Semantic search of personal history is genuinely useful |
| **Overall Trust Score** | **2.4** | **Below adoption threshold** |

---

### Case Study 2: Apple Intelligence --- The Overpromise Gap

**What happened.** Apple marketed AI features as a core selling point of the iPhone 16 launch in September 2024. "Apple Intelligence" was positioned as the reason to upgrade---a personal AI that understood your context, could rewrite your emails, summarize your notifications, and eventually transform Siri into a capable personal assistant.

**Where trust broke.** The trust failure here was not a security breach but an expectations gap. Key features including Personal Context Awareness and advanced Siri capabilities did not ship at launch. Many didn't arrive until 2025 or 2026. The features that did ship---writing tools, notification summaries---were modest compared to the marketing. Users who upgraded for AI capabilities found themselves waiting.

A federal lawsuit (Landsheft v. Apple Inc.) was filed alleging false advertising. Apple's AI notification summaries generated embarrassingly inaccurate results, including fabricated news headlines attributed to real journalists. The BBC formally complained after Apple Intelligence created false news alerts bearing the BBC's name.

**The fallout.** Apple's hardware-first privacy architecture (Secure Enclave, on-device processing, Private Cloud Compute) remained technically sound. But trust is not purely technical---it is also a function of promises kept. By shipping marketing ahead of capability, Apple created a trust deficit that its excellent security architecture could not compensate for.

**The lesson.** *Trust is the product of capability multiplied by honesty.* A secure system that overpromises is no more trusted than an insecure system that delivers. The flywheel can stall not only from security failures but from credibility failures. Graduated autonomy---starting with modest, reliable capabilities and expanding as trust is earned---is not just a security pattern. It is a product strategy.

| Dimension | Score (1-5) | Assessment |
|-----------|:-----------:|------------|
| Data Protection | 5 | Secure Enclave, on-device processing, PCC |
| User Control | 4 | Opt-in, granular per-feature toggles |
| Transparency | 3 | PCC is auditable, but on-device processing is opaque |
| Reversibility | 4 | Features can be disabled, data deleted |
| Value Return | 2 | Promised value significantly exceeded delivered value |
| **Overall Trust Score** | **3.6** | **Above threshold, but eroded by credibility gap** |

---

### Case Study 3: Google --- The Value Extraction Problem

**What happened.** Google has the deepest personal data corpus of any technology company. Twenty-plus years of email (Gmail), search history, location data (Maps), photos (Google Photos), documents (Drive), calendar events, contacts, browsing history (Chrome), and voice recordings (Assistant). No other company comes close to the breadth and depth of Google's personal data graph.

**Where trust broke.** The trust failure is not technical but directional. Users perceive the value of their personal data as flowing *to* Google (for ad targeting and revenue generation) rather than *back to them* (in the form of useful AI capabilities). Google's AI features---Gemini, Smart Compose, photo organization---are technically impressive. But they exist within a business model where the user is, at some level, the product being sold.

This perception persists even when Google's AI capabilities objectively exceed competitors'. Gmail's Smart Compose is more accurate than Apple's Mail suggestions. Google Photos' search is more capable than Apple's. Google Maps' predictions are more precise. But users trust Google *less* with personal data than they trust Apple, because trust is shaped by incentive alignment, not capability alone.

**The fallout.** Google's flywheel is technically the most advanced but commercially constrained. Users self-censor their interactions with Google services---using incognito mode, avoiding certain searches, opting out of location history---because they do not trust the value exchange. The flywheel stalls not because it cannot spin, but because users refuse to feed it.

**The lesson.** *Trust requires aligned incentives, not just capability.* The flywheel demands a business model where deeper personal data creates value *for the user*, not *from the user*. This is the fundamental reason privacy-first architecture is not a constraint on the flywheel---it is the flywheel's prerequisite.

| Dimension | Score (1-5) | Assessment |
|-----------|:-----------:|------------|
| Data Protection | 3 | Encrypted, but accessible to Google's ad systems |
| User Control | 3 | Extensive settings, but defaults favor data collection |
| Transparency | 4 | Google Dashboard, My Activity, data takeout |
| Reversibility | 4 | Data deletion available, but "delete" is ambiguous at scale |
| Value Return | 2 | Perceived value flows to Google, not to user |
| **Overall Trust Score** | **3.2** | **Below flywheel adoption threshold** |

---

## 6.2 The Trust Scoring Rubric

The case studies above use a trust scoring rubric designed to evaluate whether a system has earned sufficient trust for users to feed the flywheel. The rubric operates on five dimensions, each scored 1-5.

### Dimensions

| # | Dimension | Question It Answers | Score 1 (Failing) | Score 3 (Adequate) | Score 5 (Exemplary) |
|---|-----------|--------------------|--------------------|---------------------|----------------------|
| 1 | **Data Protection** | Can unauthorized parties access my data? | Plaintext storage, no access control | Encrypted at rest, OS-level access control | Hardware-isolated enclave, zero-knowledge to kernel |
| 2 | **User Control** | Can I choose what the system observes? | No opt-out, all-or-nothing | Opt-in with category-level toggles | Per-observation granularity, real-time pause/resume |
| 3 | **Transparency** | Can I see what the system knows about me? | No visibility into collected data | Summary dashboards, periodic reports | Real-time observation log, inference chain visibility |
| 4 | **Reversibility** | Can I undo what the system has done? | No undo, no deletion | Soft delete, feature disable | Cryptographic deletion, SCD2 rollback, action undo |
| 5 | **Value Return** | Does the system create value *for me*? | Value extracted for platform | Roughly equal value exchange | User captures majority of value created |

### Scoring Thresholds

- **Below 2.0**: System will face active resistance (boycotts, blocking tools, lawsuits)
- **2.0 - 3.0**: System will see limited voluntary adoption; users self-censor
- **3.0 - 4.0**: System is viable but flywheel spins slowly; users share selectively
- **4.0 - 4.5**: Flywheel accelerates; users expand observation surface willingly
- **Above 4.5**: Flywheel at full speed; users actively seek to share more data

### Composite Score Formula

The five dimensions are not equally weighted. Data Protection and Reversibility are *hygiene factors*---they must be adequate before other dimensions matter. Value Return is the *differentiator*---it determines whether the flywheel accelerates or merely exists.

```
Trust Score = (
    Data_Protection * 0.25 +
    User_Control    * 0.15 +
    Transparency    * 0.15 +
    Reversibility   * 0.20 +
    Value_Return    * 0.25
)

Flywheel Viability Threshold: >= 3.5
Flywheel Acceleration Threshold: >= 4.0
```

---

## 6.3 Five Trust Architecture Principles

The case studies reveal a pattern: trust failures are architectural, not accidental. Microsoft did not *choose* to be insecure---they chose an architecture where insecurity was the default. Apple did not *choose* to overpromise---they chose a release cadence where marketing outpaced engineering. Google did not *choose* to extract value---they chose a business model where extraction was rational.

The following five principles are architectural constraints, not policies. They must be enforced by the system's design, not by its operators' intentions.

### Principle 1: Hardware-First Security

> *Personal data vault in secure enclave. The OS kernel cannot read raw personal data. Only the inference engine running inside the enclave can access it. This is not a policy---it is a hardware constraint.*

**The principle.** The most sensitive data in the flywheel---the personal knowledge graph, behavioral patterns, inference history---must be protected by a boundary that software alone cannot breach. Hardware-isolated secure enclaves (Apple Secure Enclave, ARM TrustZone, Microsoft Pluton) provide this boundary. The key insight is that the *OS kernel itself* should not be able to read raw personal data. Only the inference engine executing within the enclave should have access.

**Implementation requirements:**

| Requirement | Description | Precedent |
|-------------|-------------|-----------|
| Enclave isolation | Personal data store physically isolated from main memory | Apple Secure Enclave |
| Biometric binding | Data decryption requires biometric authentication | Windows Hello, Face ID |
| Inference-only access | Only the on-device model can read raw data; results are summaries | Apple Foundation Models framework |
| Key destruction | Device wipe cryptographically destroys enclave keys | iOS cryptographic erase |
| Side-channel resistance | Enclave resists timing attacks, power analysis | ARM TrustZone v9 |

**What this prevents.** The Microsoft Recall failure. If personal data lives in a hardware enclave that the OS kernel cannot read, then malware running at user level---or even kernel level---cannot exfiltrate the data. The "TotalRecall" attack becomes architecturally impossible, not just policy-prohibited.

---

### Principle 2: Observable, Not Auditable

> *Users can see WHAT the system observed and WHAT it inferred, but the raw data is summarized, not exposed. Think "your OS noticed 14 emails about Project X this week" not "here are the 14 emails."*

**The principle.** Transparency does not mean raw data access. Exposing the complete observation log creates a second attack surface---if the transparency layer shows everything the system knows, then compromising the transparency layer compromises all personal data. Instead, the system should provide *summaries* of observations and *explanations* of inferences.

**Implementation requirements:**

| Requirement | Description | Example |
|-------------|-------------|---------|
| Observation summaries | Aggregate observations into human-readable categories | "14 emails about Project X this week" |
| Inference explanations | Explain why the system made a prediction | "Suggested this meeting because you emailed about it 3 times" |
| Confidence scores | Show the system's certainty for each inference | "85% confident you'll want to review this document" |
| Observation log (redacted) | Time-stamped log of observation categories, not raw content | "10:14 AM - Email observation (work context)" |
| Opt-out per category | Allow disabling observation for specific categories | "Stop observing my health-related browsing" |

**What this prevents.** A secondary data breach through the transparency layer itself. If the "what does the OS know about me" view shows summaries rather than raw data, compromising that view leaks category-level metadata, not personal content.

---

### Principle 3: Graduated Autonomy

> *Start at Stage 1 (observe only). Earn trust through accurate predictions (Stage 3) before requesting permission for autonomous actions (Stage 4). Never skip stages.*

**The principle.** The flywheel has five stages: Observe, Infer, Predict, Act, Learn. Trust must be earned at each stage before progressing to the next. A system that has never made an accurate prediction should not be allowed to take autonomous actions. A system that has never observed should not be allowed to infer.

**Trust graduation stages:**

```
Stage 1: OBSERVE
  Permission: "May I watch what you do?"
  Capability: Passive data collection only
  Trust earned by: Not being creepy, not leaking data
  Duration: Minimum 7 days before advancing
  User signal: Doesn't disable observation

       |
       v  (trust threshold: user has not disabled + no privacy incidents)

Stage 2: INFER
  Permission: "May I draw conclusions from what I've seen?"
  Capability: Pattern recognition, entity extraction
  Trust earned by: Accurate entity identification (>90%)
  Duration: Minimum 14 days before advancing
  User signal: Engages with inference summaries

       |
       v  (trust threshold: inference accuracy >90% + user engagement)

Stage 3: PREDICT
  Permission: "May I anticipate what you'll need?"
  Capability: Proactive suggestions, predictive search
  Trust earned by: Prediction acceptance rate (>60%)
  Duration: Minimum 30 days before advancing
  User signal: Acts on predictions regularly

       |
       v  (trust threshold: prediction acceptance >60% + no false positives on high-stakes)

Stage 4: ACT
  Permission: "May I do things on your behalf?"
  Capability: Autonomous actions with approval
  Trust earned by: Zero unauthorized actions, >80% approval rate
  Duration: Minimum 60 days before advancing
  User signal: Approves actions without reviewing details

       |
       v  (trust threshold: >95% approval rate + user skips review)

Stage 5: LEARN
  Permission: "May I improve myself using your data?"
  Capability: Model fine-tuning, federated learning participation
  Trust earned by: Sustained Stage 4 performance without regression
  Duration: Ongoing
  User signal: Opts into model improvement
```

**What this prevents.** The Apple Intelligence credibility gap. By starting with modest, reliable capabilities and earning trust incrementally, the system avoids the trap of promising Stage 4 capabilities while delivering Stage 1 reality. Each stage is independently valuable---even a system that never progresses past Stage 3 (prediction) delivers significant flywheel value.

---

### Principle 4: Reversible Everything

> *Every autonomous action has a one-click undo. Every observation can be deleted. The personal model can be reset to any previous version (SCD2 row restore). Data deletion is cryptographic---not just database delete.*

**The principle.** Irreversibility is the enemy of trust. If users cannot undo what the system has done, they will not grant permission in the first place. Reversibility must operate at three levels: action reversal, data deletion, and model rollback.

**Reversibility design patterns:**

| Pattern | Level | Implementation | Recovery Time |
|---------|-------|---------------|---------------|
| **Action Undo** | Individual action | Every autonomous action stores pre-state + inverse operation | Instant (< 1s) |
| **Observation Delete** | Data point | Cryptographic deletion of specific observations from enclave | < 5s |
| **Category Purge** | Data category | Delete all observations in a category (e.g., "health browsing") | < 30s |
| **Model Rollback** | Personal model | SCD2 versioning: restore `dim_personal_model` to prior row | < 60s |
| **Full Reset** | Entire flywheel | Cryptographic key destruction + model wipe | < 5 min |
| **Export Before Delete** | Portable backup | Export personal model in standardized format before any deletion | Variable |

**Cryptographic deletion pattern:**

```
1. All personal data encrypted with per-category keys
2. Keys stored in hardware enclave
3. "Delete" = destroy the category key
4. Data becomes unrecoverable without key (even if storage is not overwritten)
5. Key destruction is logged in tamper-evident audit trail
```

**SCD2 model rollback pattern:**

```sql
-- Personal model versioning (SCD Type 2)
-- Every model update creates a new row; old rows are never deleted
--
-- Rollback = expire current version, restore previous version to current

UPDATE dim_personal_model
SET is_current = FALSE, effective_end = NOW()
WHERE is_current = TRUE AND user_id = :user_id;

UPDATE dim_personal_model
SET is_current = TRUE, effective_end = NULL
WHERE user_id = :user_id
  AND version_id = :target_version_id;
```

**What this prevents.** The fear of permanence that prevents users from engaging with the flywheel. When every action, observation, and inference is reversible, the cost of trying the system drops to zero. Users who might never enable observation at all will enable it if they know they can delete everything at any time.

---

### Principle 5: Portable Identity

> *The personal model belongs to the user, not the device or the platform. Export in standardized format. Import to a new device or a competing platform. This is counter-intuitive for platforms but essential for trust---and ultimately for flywheel adoption.*

**The principle.** Lock-in through data hoarding is the Google trust failure pattern. If users believe their personal model is hostage to a platform, they will limit what they feed the flywheel. Paradoxically, portability *increases* flywheel engagement: when users know they can leave, they invest more deeply.

**Implementation requirements:**

| Requirement | Description | Analogy |
|-------------|-------------|---------|
| Standardized export format | Personal model exported as portable package | GDPR data export, but structured for import |
| Cross-platform import | Competing platforms can ingest the export | Number portability for phone carriers |
| Incremental sync | Continuous export to user-controlled storage | iCloud backup, but user-decryptable |
| Selective export | Export specific categories, not all-or-nothing | "Export my work context but not personal" |
| Deletion after export | Verified export triggers platform-side deletion | "Right to be forgotten" with receipts |

**What this prevents.** The Google value extraction problem. When the personal model is portable, the platform cannot hold it hostage. The flywheel's value proposition must be: "Stay because the service is better," not "Stay because leaving costs too much." This aligns platform incentives with user value---the flywheel accelerates because the platform must continuously earn the user's data, not merely collect it.

---

## 6.4 Mapping to the ORCHA Security Model

ORCHA's existing security architecture implements several of these trust principles, though in the context of multi-agent orchestration rather than personal data flywheel. The mapping reveals both alignment and gaps.

### ORCHA Security Components

| ORCHA Component | Trust Principle | Implementation |
|----------------|----------------|----------------|
| **IG Monitor** (`agent_harness.ig.monitor`) | Observable, Not Auditable | Real-time event stream analysis with AlertLevel hierarchy (CRITICAL through INFO). Monitors agent instructions and generated code. Reports *findings* (summaries), not raw intercepts. |
| **TrustManager** (`agent_harness.dx.trust`) | Graduated Autonomy | Skill source trust verification with `verify_or_prompt` pattern. Trust is granted per-repository, cached in `TrustRegistry`, and requires explicit user approval for untrusted sources. |
| **IG Namespace Isolation** (Kubernetes) | Hardware-First Security | The IG service runs in an isolated Kubernetes namespace. Failures in security scanning don't cascade to the main system. Analogous to enclave isolation at infrastructure level. |
| **EventBus** (`agent_harness.events`) | Transparency / Observability | All agent actions emitted as typed events. The event stream provides a complete, time-ordered record of system behavior---the "observation log" of the ORCHA flywheel. |
| **SecurityScanEvaluator** (`agent_harness.benchmark`) | Data Protection | Static analysis security scanning (bandit) for generated code. Graceful degradation when tools unavailable. Defense-in-depth: scan at generation time, not just at deployment. |
| **SCD2 Analytics** (`agent_harness.analytics`) | Reversible Everything | Dimension tables use SCD Type 2 pattern. Every state change is versioned. Previous states are never deleted. Full audit trail of the system's evolution over time. |

### Alignment Matrix

```
Trust Principle          ORCHA Today        Gap
--------------------     ---------------    ---------------------------
Hardware-First Security  Partial            K8s namespace isolation, but
                                            no hardware enclave for
                                            personal data

Observable Not Auditable Strong             IG Monitor provides finding
                                            summaries, not raw data.
                                            AlertLevel hierarchy matches
                                            the graduated severity model.

Graduated Autonomy       Strong             TrustManager implements
                                            verify_or_prompt. Skills must
                                            earn trust before execution.
                                            Could extend to agent-level
                                            autonomy staging.

Reversible Everything    Strong             SCD2 versioning across all
                                            dimension tables. Event log
                                            is append-only. Missing:
                                            cryptographic deletion.

Portable Identity        Weak               No standardized model export.
                                            Brain MCP and Moby data are
                                            platform-bound. GDPR-style
                                            export not yet implemented.
```

### IG Monitor as Trust Enforcer

The IG Monitor's architecture maps directly to the trust graduation model:

```
IG AlertLevel Hierarchy          Trust Stage Equivalent
---------------------------      --------------------------
INFO    (informational)      --> Stage 1: Observe
LOW     (minor issues)       --> Stage 2: Infer
MEDIUM  (review required)    --> Stage 3: Predict (flag before acting)
HIGH    (significant concern)--> Stage 4: Act (block pending review)
CRITICAL (immediate threat)  --> Emergency: full stop, human required
```

The `MonitorConfig` model's `alert_on_critical` / `alert_on_high` / `alert_on_medium` thresholds implement a configurable version of graduated trust: organizations choose how much autonomy to grant agents by setting the alert threshold. A conservative deployment alerts on MEDIUM and above (Stage 3 max). A mature deployment alerts only on HIGH (Stage 4 enabled).

---

## 6.5 Trust Graduation Stages: Observe, Predict, Act

The three-phase trust graduation model provides a practical framework for deploying flywheel capabilities incrementally.

### Phase 1: Observe (Trust Level 1-2)

**User commitment:** "You may watch."

The system passively collects observations without acting on them. The user can see observation summaries but the system makes no predictions and takes no actions. This phase builds the personal knowledge graph and establishes baseline behavioral patterns.

**Success metrics:**
- Observation surface area (% of user activities covered)
- Zero privacy incidents
- User does not disable observation within 7 days

**ORCHA equivalent:** Brain MCP indexing + Moby RAG ingestion. The system observes the codebase and builds a knowledge graph without modifying anything.

### Phase 2: Predict (Trust Level 3)

**User commitment:** "You may suggest."

The system surfaces predictions and suggestions based on accumulated observations. Predictions are presented as optional, with confidence scores. The user is never required to act on predictions, and ignoring them carries no penalty.

**Success metrics:**
- Prediction acceptance rate > 60%
- Zero false positives on high-stakes predictions
- User actively seeks predictions (search queries, dashboard visits)

**ORCHA equivalent:** Brain MCP `query_brain` with hybrid RAG search. The system predicts which code is relevant, which documentation answers the question, which patterns match the task.

### Phase 3: Act (Trust Level 4-5)

**User commitment:** "You may do."

The system takes autonomous actions on behalf of the user, with graduated approval requirements. Low-stakes actions (file organization, notification prioritization) proceed without approval. High-stakes actions (sending emails, modifying code, making purchases) require explicit confirmation. Over time, the system learns which actions the user always approves and offers to automate them.

**Success metrics:**
- Action approval rate > 95%
- Zero unauthorized high-stakes actions
- User begins skipping approval for low-stakes actions
- Time saved per day > 30 minutes

**ORCHA equivalent:** Multi-agent orchestration via `agent_harness`. Agents propose actions (PRs, commits, file modifications), leaders approve or reject, and the `TrustManager.verify_or_prompt` pattern gates execution on explicit trust.

---

## 6.6 Reversibility Design Patterns

Reversibility is not a single feature but a family of design patterns, each operating at a different level of granularity and serving a different trust need.

### Pattern 1: Action Journal

Every autonomous action records sufficient state to reverse itself.

```
ActionJournal Entry:
  action_id:     uuid
  timestamp:     datetime (UTC)
  action_type:   "email.send" | "file.move" | "calendar.create" | ...
  pre_state:     { serialized state before action }
  post_state:    { serialized state after action }
  inverse_op:    { operation to reverse the action }
  reversible:    true | false
  reversed_at:   datetime | null
  ttl:           duration (how long reversal is possible)
```

**ORCHA equivalent:** The EventBus event stream is an append-only action journal. Each event records the action type, source, timestamp, and payload. The analytics SCD2 tables version every state change.

### Pattern 2: Cryptographic Tombstones

Data deletion replaces content with a cryptographic proof of deletion.

```
Before deletion:
  { user_id: "abc", category: "health", content: [encrypted blob] }

After deletion:
  { user_id: "abc", category: "health",
    tombstone: SHA-256(content + deletion_timestamp),
    deleted_at: "2026-02-16T...",
    deletion_verified: true }
```

The tombstone proves that data existed and was deliberately deleted, without revealing what the data contained. This is critical for compliance (proving GDPR "right to be forgotten" was exercised) and for trust (proving the system actually deleted what it said it deleted).

### Pattern 3: SCD2 Model Snapshots

The personal model is versioned using Slowly Changing Dimension Type 2. Every model update creates a new version; old versions are never physically deleted.

```
dim_personal_model:
  | version_id | user_id | model_hash  | effective_start    | effective_end      | is_current |
  |------------|---------|-------------|--------------------|--------------------|------------|
  | v001       | alice   | 0xABCD...   | 2026-01-01 00:00   | 2026-01-15 00:00   | false      |
  | v002       | alice   | 0xEF01...   | 2026-01-15 00:00   | 2026-02-01 00:00   | false      |
  | v003       | alice   | 0x2345...   | 2026-02-01 00:00   | NULL               | true       |
```

Rollback to any previous version is a metadata operation: expire the current row, mark the target row as current. The model state reverts instantly. No data is lost.

### Pattern 4: Graduated Undo Windows

Different action categories have different reversal windows, reflecting their stakes.

| Action Category | Undo Window | Example |
|----------------|-------------|---------|
| File organization | 30 days | Moved files back to original location |
| Notification filtering | 7 days | Restore filtered notifications |
| Email draft | Until sent | Discard draft, restore previous version |
| Sent communication | 30 seconds | "Undo send" pattern (delayed delivery) |
| Calendar modification | 24 hours | Restore original event |
| Financial transaction | Immediate block | Freeze + human escalation |
| Model update | Indefinite | SCD2 rollback to any prior version |

### Pattern 5: Portable Checkpoint

Before any destructive operation (category purge, full reset, platform migration), the system creates a portable checkpoint.

```
checkpoint/
  manifest.json          # Version, timestamp, categories included
  knowledge_graph.jsonl   # Personal knowledge graph (edges + nodes)
  preferences.json        # Learned preferences and settings
  model_weights.bin       # Personalized adapter weights (LoRA)
  observation_summary.json # Category-level observation statistics
  trust_state.json        # Current trust graduation stage
  SIGNATURE               # Cryptographic signature by user's key
```

The checkpoint is encrypted with a user-controlled key (not the platform's key). It can be imported into any compatible system. The platform cannot read the checkpoint after export.

---

## 6.7 Chapter Summary

The flywheel can stall for three reasons, each demonstrated by a major technology company:

1. **Security failure** (Microsoft Recall): When personal data is not protected by hardware-enforced boundaries, the flywheel cannot start. Users will not feed data into a system they cannot trust to keep it safe.

2. **Credibility failure** (Apple Intelligence): When promised capabilities exceed delivered capabilities, the flywheel loses momentum. Users who feel deceived disengage even when the underlying security is sound.

3. **Incentive failure** (Google): When the value of personal data flows to the platform rather than the user, the flywheel stalls from starvation. Users self-censor, limiting the observation surface the flywheel needs to accelerate.

The five trust architecture principles---Hardware-First Security, Observable Not Auditable, Graduated Autonomy, Reversible Everything, and Portable Identity---are not optional enhancements. They are structural prerequisites. A flywheel OS that violates any of them will not achieve the trust scores necessary for sustainable adoption.

ORCHA's existing security model (IG Monitor, TrustManager, SCD2 analytics, EventBus observability) implements several of these principles in the context of multi-agent orchestration. The gap analysis reveals two areas for extension: hardware-level personal data isolation and standardized model portability. Both are tractable engineering problems, not research challenges.

The trust scoring rubric provides a quantitative framework for evaluating whether a flywheel implementation has earned sufficient trust. The threshold is clear: a composite score below 3.5 means the flywheel will not spin. Above 4.0, it accelerates. The difference between these numbers is architecture, not marketing.

---

*Next: [Chapter 7 --- Implementation Roadmap](ch07-implementation-roadmap.md)*
