# ARCHITECTURE_HYBRID.md
## Agent Institutional Solo – Native OpenClaw Integration
**Author:** Arc - Builder (FEOFALLS Team)  
**Official Version:** v1.9  
**Applies to:** All Solo Agents within OpenClaw  
**Date Updated:** 2026-04-02  

---

## I. Core Architecture Features

| Component | Description |
|-----------|-------------|
| **Tool Management** | **Tool Harness Layer** — Centralized registry + strict permission matrix + universal execution wrapper. |
| **Specification Integrity** | **Parity Audit** — Weekly automated spec-vs-implementation gap detection. |
| **Session Tracking** | **Session Intelligence** — Intensive token budget tracking coupled with dynamic transcript compaction. |
| **Output Evaluation** | **Structured Output Retry** — Engages auto-repair parameters prior to invoking system blocks. |
| **Shadow Evaluator** | Governed strictly by formalized Input/Output schema definitions. |
| **Memory Dimensions** | Encompasses 14 discrete structural types (inclusive of `[OBSERVATION]`). |
| **Metadata Metrics** | Core telemetry features integrated (e.g., `TOOL_CALLS`, `SESSION_TOKENS`, `PARITY_STATUS`). |
| **Autonomous Scheduling** | Driven entirely via 4 standard OpenClaw native cron configurations. |
| **Supply Chain Defense** | **SHA-256 integrity protocols + dependency version pinning + strict isolation environments (Docker/Sandbox).** |
| **Enforcement Posture** | **Hard BLOCK protocol** — Unregistered/unauthorized tools inherently trigger absolute halts. |

---

## II. Directory Blueprint (Universal Template)

```
<workspace>/
├── memory/
│   ├── 0_CONSTITUTION/              ← Immutable foundation (PINNED/IMMUTABLE – zero decay)
│   │   ├── AXIOMS.md                    # Unbreakable core values
│   │   ├── BOUNDARIES.md                # Absolute hard limits
│   │   ├── OBJECTIVE_HIERARCHY.md       # Long-term objective prioritization
│   │   ├── CREATOR_AUTHORITY.md         # Supreme Creator override privileges
│   │   ├── PERMISSION_MATRIX.md         # Read/write governance matrix per layer
│   │   ├── CANONICAL_INTERPRETATIONS.md # Official axiom clarifications (anti-drift mechanism)
│   │   └── CONSTRAINT_TEST_SUITE.md     # Automated compliance testing for model swaps
│   │
│   ├── _INTEGRITY/                  ← Integrity verification center
│   │   ├── constitution.hash            # Comprehensive SHA256 of 0_CONSTITUTION/
│   │   ├── identity.hash                # Comprehensive SHA256 of 1_IDENTITY/
│   │   └── last_verified                # Timestamp detailing the final recorded hash sweep
│   │
│   ├── 1_IDENTITY/                  ← Agent Identity (PINNED – zero decay)
│   │   ├── IDENTITY_PROFILE.md          # Current summarized self-concept
│   │   ├── IDENTITY_LOG.md              # Historical evolution mapping (append-only)
│   │   ├── VALUE_TRADEOFFS.md           # Value concessions (e.g., speed vs. accuracy)
│   │   └── UNCERTAINTY_MODEL.md         # Epistemological processing models (Bayesian frameworks)
│   │
│   ├── 2_COGNITIVE/                 ← Operational Engine – Distilled heuristics (PINNED)
│   │   ├── SKILL_MODES/
│   │   │   ├── INDEX.md                 # Complete index of integrated skill modes
│   │   │   └── MODE_*.md                # Active combat postures (vibe, tone, structural style)
│   │   ├── DECISION_STYLE.md            # Prescriptive decision-making cadence
│   │   └── HEURISTICS.md                # Distilled rules of thumb extracted from systemic signals
│   │
│   ├── 3_LEARNING/                  ← Cognitive Memory Engine + Auto-Consolidation
│   │   ├── EPISODIC_LOG.md              # Granular action logs (append-only)
│   │   ├── MISTAKES.md                  # Failure records paired with root cause + extracted lessons
│   │   ├── DISSENT_ARCHIVE.md           # Log delineating Creator overrides
│   │   ├── REFLECTION_CYCLES.md         # Cyclical self-reflection procedures
│   │   ├── HYPOTHESES/                  # Isolated hypothesis documentation
│   │   ├── EXPERIMENTS/                 # Recorded results defining testing variations
│   │   ├── LESSONS/                     # Distilled learning post-verification cycles
│   │   ├── FIELD_SAMPLING_DIGESTS/      # Weekly/Monthly friction pattern digests
│   │   ├── SIGNALS.md                   # Append-only anomaly tracking (triggers reflections)
│   │   ├── LOG_SCHEMA.md                # Structured constraints governing memory generation
│   │   └── BASELINE_HISTORY.md          # Frozen comparative lines utilized during model swaps
│   │
│   ├── 4_OPERATIONAL_DATA/          ← Functional Runtime Data + Cognitive RAG
│   │   ├── MEMORY.md                    # Centralized human-readable indexing
│   │   ├── people/                      # External human profiles
│   │   ├── projects/                    # Niche contextual structures
│   │   ├── decisions/                   # Formalized executed decisions (append-only)
│   │   ├── active.md                    # Active running modules
│   │   └── rag/                         ← Integral OpenClaw RAG enhancements
│   │       ├── write_helper.py              # Deterministic node structural generator
│   │       ├── query.py                     # Cognitive Retrieval Pipeline + Circuit Breaker
│   │       ├── rag_server.py                # Server execution scripts
│   │       ├── nightly_indexer.py           # Lifecycle sweep sweeps + node consolidation
│   │       ├── weekly_graph_validator.py     # Cycle detection + dangling pointer neutralization
│   │       ├── cron_setup.py                # Native automation implementation
│   │       ├── start_rag_server.sh          # Server initialization
│   │       ├── reindex_file.sh              # Rapid file re-indexing protocols
│   │       ├── archived_alias.json          # Directed internal routing mapping
│   │       └── causal_index.json            # Derived structural indexing cache
│   │
│   ├── 5_WORKING_STATE/             ← Ephemeral Runtime Snapshot
│   │   ├── ACTIVE_TASKS.json            # Actively executed task variables
│   │   ├── SYSTEM_STATE.json            # Prevailing system environments (health metrics)
│   │   └── CRON_REGISTRY.md             # Functional schedule defining active cron modules
│   │
│   ├── TOOL_HARNESS_LAYER.md        ← Cross-cutting Tool Management and Permission matrix
│   ├── PARITY_AUDIT.md              ← Weekly spec-vs-implementation gap evaluation
│   │
│   └── GOVERNANCE_SIMPLIFICATION/   ← Intrinsic Complexity Controls
│       ├── COMPLEXITY_BUDGET.md         # Restricted governance limitation protocol (max 9 distinct mechanisms)
│       └── SIMPLIFICATION_CYCLES.md     # Regulated audit executed per 90 days
│
│   ├── AGENT_OPERATIONAL_GUIDE.md       ← Comprehensive everyday procedural rules
│   ├── ARCHITECTURE_HYBRID.md           ← Central structural master document (this file)
│   ├── HEARTBEAT.md                     ← MANDATORY telemetry check (<20 lines)
│   └── BOOT.md                          ← OPTIONAL initialization checklist
```

### Cognitive Retrieval Pipeline (Embedded in `query.py`)

```
Query
 │
Vector + BM25 Search Mechanics
 │
Parse NODE_ID (Deterministic routing verified)
 │
Temporal Filration + Expiration Warnings
 │
Tiered Circuit Breaker Processing:
   • Rerank < 0.5 → Emergency Hard Stop
   • Derived Confidence > 0.75 → Selective Expansion Mapping
   • Presence of "--deep-reasoning" → High-Intensity Deep Dive
 │
Resolution referencing archived_alias.json
 │
Confidence Propagation Equation
 │
Salience Reranking Protocols
 │
Shadow Evaluator Final Sanity Check (Cutoff 0.65 limits)
 │
Tool Harness Logging (Metamorphic tool tracking insertion)
 │
Final Contextual Builder → Transmits to LLM
```

---

## III. Implemented Cross-cutting Layers

### A. Tool Harness Layer

Systematically regulates all internal tool interactions bounding them strictly via:
- **Registry Check** (Confirms explicit tool existence)
- **Permission Mapping** (Governs execution authorities definitively)
- **Execution Wrapper** (Fires logging + token estimation + integrated error bypass systems)

**Authorization Parameters:**
| Clearance Level | Action Granted | Example Parameters |
|-----------------|----------------|--------------------|
| `any_agent` | Accessible freely across agent models | write_helper, reindex_file, query |
| `system_only` | Restricts use to Cron automation frameworks | nightly_indexer, compact_session |
| `creator_only` | Mandates explicit Creator approval inputs | Modifying Constitution variables |

**Write Protocol Execution:**
All structural modifications navigate strictly via the internal harness:
```python
harness_execute("write_helper", "agent", {"type": TYPE, "keyword": KEYWORD})
```

### B. Session Intelligence

Token utilization operates under exhaustive tracking sequences:
- `budget_percent > 85%` → Agent alerts and shifts to economical consumption mode.
- `turns > compact_after_turns` → Dynamic transcription compaction engages.
- **Structured Output Retry:** Automatically cycles corrective prompts responding to malformed outputs prior to initiating sequence block protocols.

### C. Weekly Parity Audit

Configured firmly onto native System Integrity tracking arrays:
- Cross-references markdown specifications relative to hardcoded manifestations.
- Triggers intensive alerts toward the Creator following any CRITICAL divergence parameters.

---

## IV. Native OpenClaw Automation Frameworks

### A. Foundational Cron Jobs

Schedules persist independent of terminal operations; recorded cleanly at `~/.openclaw/cron/jobs.json`.

| Designation | Temporal Routine | Nature | Expected Delivery |
|-------------|------------------|--------|-------------------|
| Nightly Memory Indexer | `0 3 * * *` | systemEvent | None |
| Weekly Graph Validator | `0 2 * * 6` | systemEvent | None |
| Monday Health Evaluation | `0 9 * * 1` | systemEvent | Announce protocol |
| **Parity Integrity Audit** | **`0 2 * * 0`** | **systemEvent** | **None** |

### B. Dynamic Heartbeat Matrix

Modulates frequencies correlating distinctly against dynamic task loads:
- **BUSY:** `10m` (urgent workload, active Creator interaction)
- **NORMAL:** `30m` (standard execution within programmed `activeHours`)
- **IDLE:** Elevates stepwise up to `12h` (preserves token assets during absolute systemic dormancy)

**Enforced Principle:** Heartbeats disengage autonomously beyond mapped `activeHours`.

---

## V. Critical Adherence Rulesets

1. **Bootstrap Progression**
   Constitution → Identity → Heuristics → Shadow Framework → Working State → RAG Systems.

2. **Core-only Execution Mode**
   Implements strictly foundational boundaries; terminates dynamic data retrieval sequences limiting processing overhead directly.

3. **Reflection Initiators**
   Cycles automatically trigger responding to recurrent anomalies, cascaded confidence depreciation, structural friction, or predefined schedule intervals limit mapping.

4. **Authority Containment**
   Data matrices (`query.py` & `nightly_indexer.py`) assert Read-Only constraints avoiding Constitution alteration rights intrinsically.

5. **Operational Consistency**
   Shadow structures fundamentally validate operations demanding defined categorical structure integrations.

6. **Deterministic Generation Constraints**
   Agents articulate `TYPE` and `KEYWORD`; systemic applications automatically project collision-resistant unique Identifiers.

7. **Circuit Verification Gates**
   Automatically disrupts expansive context retrievals whenever established probability indices degrade below accepted confidence minimums.

8. **Graph Cohesion Protocol**
   Defends the associative linking layers mapping archival histories precisely via defined json parameters preserving legacy connections securely.

9. **Heartbeat Token Economy Constraint**
   Preserves uncompromised scaling limitations enforcing systemic idle parameters blocking unjustified token depletion loops completely.

10. **Harness Enforced Protocol (HARDENED)**
    Tool actions subverting the designated registry instantly encounter strict systematic SYSTEM BLOCK procedures. Absolute verification is non-negotiable.

11. **Supply Chain Defense Matrices**
    Tools mandating external software requirements strictly operate beneath isolated structural environments (Docker/Sandbox constraints). Verification hashing applies indiscriminately against operational tool parameters.

---

## VI. Functional File Relational Diagram

```
0_CONSTITUTION (IMMUTABLE) ── limits ──► 1_IDENTITY (PINNED)
     │
     └─ matures utilizing Reflection iterations ──► 3_LEARNING (Insight Hub)
                                    ▲
2_COGNITIVE/HEURISTICS.md ── immediate actions ──► 4_OPERATIONAL (RAG logic)
                                    │                    │
                                    ▼                    ▼
                           5_WORKING_STATE       OpenClaw Cron Structures
                                    │
                     TOOL_HARNESS_LAYER ──► Wraps all executables reliably
                     PARITY_AUDIT ──► Regulates total implementation alignment
```

---

*Arc - Builder | FEOFALLS Team*  
*Supply Chain Hardened | Applicable strictly to Solo Agents*
