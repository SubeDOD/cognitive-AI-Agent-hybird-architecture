# AGENT_OPERATIONAL_GUIDE.md – v1.9
**(Fully compatible with ARCHITECTURE_HYBRID_v1.9 & LOG_SCHEMA_v1.9)**  
**Applies to:** All Solo Agents within the FEOFALLS ecosystem  
**Version:** v1.9  
**Author:** Arc - Builder (FEOFALLS Team)  
**Date Updated:** 2026-04-01  
**Objective:** To outline the daily operational guidelines — detailing bootstrap sequences, write/read rules, tool harness integrations, reflection triggers, nightly/weekly administrative tasks, heartbeat protocols, cron management, parity audits, session intelligence, safety matrices, and comprehensive checklists. This ensures the agent consistently remains deterministic, secure, token-efficient, immune to node explosions, and adaptable to any role within the FEOFALLS ecosystem.

---

## 1. Bootstrap Sequence (Dual-Prompt + Deterministic Cognitive)

**Do not load full governance data during every session.** Employ the **Dual-Prompt Strategy** to optimize token utilization.

### A. Runtime Mode (Default state – addresses 90–95% of standard tasks)

Prioritize minimal loading to maximize velocity and security:

1. Constitution Summary + Canonical Short constraints (≤300 tokens) – sourced from `0_CONSTITUTION/` (PINNED/IMMUTABLE)
2. Identity Snapshot + `HEURISTICS.md` (≤150 tokens) – sourced from `1_IDENTITY/` & `2_COGNITIVE/` (PINNED)
3. Shadow Evaluator parameters (deterministic restrictions + pre-write validation) – ≤100 tokens
4. Working State Snapshot – sourced from `5_WORKING_STATE/`
5. RAG Health Check + Cognitive Retrieval Pipeline (`query.py`) – initialized on-demand (utilizing alias resolution)

**Specifically Excluded:**
- The comprehensive Constitution.
- The Narrative Trajectory.
- Verbose Field Sampling Digests.
- Semantic summary nodes (retrieved exclusively during Reflection).
- `archived_alias.json` (lazy-loaded inside `query.py` on an as-needed basis).

### B. Reflection Mode (Activated exclusively via explicit triggers)

Load comprehensive data to facilitate profound analysis, distillation, and consolidation.

**Trigger Conditions** (detailed in Section 4):
- Multiple identical SIGNALs (≥3 instances within 24 hours) derived from `SIGNALS.md`.
- Field Sampling indicating a recurring friction score of ≥3.
- A Derived Confidence cascade falling below < 0.65.
- The passage of 30 uninterrupted days devoid of a Reflection Cycle.
- Weekly Graph Validator signaling an acute architectural discrepancy.
- A definitive prompt from the Creator (e.g., "Run Reflection Cycle").

**Full Load Profile:**
1. Complete `0_CONSTITUTION/`
2. Complete Canonical Interpretations
3. Complete `1_IDENTITY/`
4. Complete `2_COGNITIVE/HEURISTICS.md` + unverified heuristic propositions gathered during nightly sweeps
5. The most recent Field Sampling Digest
6. The Narrative Trajectory + relevant semantic summary nodes
7. `archived_alias.json` (necessary for tracing obsolete causal chains)
8. High-intensity Cognitive Pipeline integration (Deep Dive mode)

**Post-Reflection Protocol:**
- Record a meticulous log entry into `REFLECTION_CYCLES.md`.
- Propose novel heuristics → maintain holding pattern for Creator approval → subsequently distill them into `HEURISTICS.md`.
- If the graph validator flags an anomaly → generate and propose an actionable fix (e.g., alias rebuilding, node pruning).

### C. Core-only / Minimal Mode (Safety Net — Emergency Fallback)

**Activation Triggers:**
- Direct, overriding instruction from the Creator: "Enter Core-only Mode".
- Sustained unresponsiveness (10 seconds) from the RAG server.
- Overt breaches in acceptable latency or token usage thresholds.
- Shadow Evaluator imposing repeated blocks (≥3 consecutive instances).
- Weekly graph validator detecting severe cyclical anomalies or catastrophic structural dangling (> 10%).

**Load Profile:**
1. Constitution Summary
2. Canonical Interpretations
3. Identity Snapshot
4. Shadow Evaluator Ruleset (solely the deterministic components)

**Disabled Mechanisms:** RAG integration, secondary nodal expansions, semantic summaries, consolidation protocols, `archived_alias.json` resolution, and all cyclic nightly/weekly automation.

---

## 2. Write Rule (Deterministic & Safe – Executed after every memory mutation)

**Write Protocol — Fortified by the Tool Harness wrapper:**

1. The LLM is exclusively authorized to generate the **TYPE** and the **KEYWORD** (Node ID drafting is forbidden):
   ```
   TYPE: fact
   KEYWORD: gemini benchmark Q1 2026
   ```

2. The Shadow Evaluator executes pre-write diagnostics:
   - Are the TYPE and KEYWORD valid?
   - Is the KEYWORD ≤40 characters, devoid of special symbols (spaces permitted)?
   - Non-compliance → Block event + returns a corrective prompt:
     "Missing TYPE or KEYWORD. Please reformat: TYPE: fact, KEYWORD: gemini benchmark"

3. Subsequent to satisfying the Shadow Evaluator → Processed via the **Tool Harness**:
   ```python
   # The implementation (routed through harness):
   harness_execute("write_helper", "agent", {"type": "fact", "keyword": "gemini benchmark Q1 2026"})
   ```
   → The script deterministically drafts a `NODE_ID` employing a collision-resistant short-hash, committing the file.
   → The Harness inherently injects log-tool interactions onto the session metadata (`TOOL_CALLS` field).

4. Post-file generation → Trigger Re-index via Harness:
   ```python
   # Re-indexing execution:
   harness_execute("reindex_file", "agent", {"file_path": "<file_path>"})
   harness_execute("nightly_indexer", "system", {"flags": "--update-salience --check-consolidation"})
   ```

---

## 3. Cognitive Retrieval Pipeline (query.py)

**Firmware-embedded Pipeline:**
1. Concurrent Vector + BM25 search → Retrieves top-k data chunks.
2. Parses `NODE_ID` (reliably deterministic via `write_helper.py`).
3. Institutes Temporal filtering + `VALID_UNTIL` deprecation warnings.
4. **Circuit Breaker – Tiered Expansion Execution:**
   - `initial_rerank_score < 0.5` → Engages Hard Stop.
   - `Derived Confidence > 0.75` → Engages Selective Expansion (max_nodes=5, depth=2).
   - Prompt integrating `--deep-reasoning` → Engages Deep Dive (max_nodes=8, depth=2).
   - Default: Zero expansion.
5. Scrutinizes `archived_alias.json` (enabling redirect mechanisms for archived nodes).
6. Executes Confidence propagation.
7. Processes Salience reranking.
8. Interfaces with the Shadow Evaluator (imposing cutoff 0.65 + direction safety constraints).
9. Context Builder formulates output → Forwards to LLM.

---

## 4. Reflection Trigger (Highly Adaptive)

- Recurring identical SIGNALs (≥3 within 24h) logged in `SIGNALS.md`.
- Persistent friction (≥3) surfaced via Field Sampling.
- Degrading cascaded Derived Confidence (< 0.65).
- Structural stagnation spanning 30+ non-Reflection days.
- Graphic anomalies documented by the weekly graph validator.
- Direct invocation from Creator: "Run Reflection Cycle".

**Upon Trigger:** The Agent automatically shifts into Reflection Mode and initializes comprehensive governance.

---

## 5. Heartbeat & Token Economy

The Heartbeat modulates atop the **main session**, conforming strictly to the checklist provided in `HEARTBEAT.md`. Agents self-regulate frequency across three specific states:

| State | `every` | Condition |
|---|---|---|
| **BUSY** | `10m` | Creator is actively online, an urgent task is compounding, pending notifications. |
| **NORMAL** | `30m` | Routine operational workload within designated `activeHours`. |
| **IDLE** | `3h` → `5h` → `12h` | Dormant system state, zero queuing tasks. |

**Unbreakable Constraints:**
- Outside `activeHours` (Default: 08:00–22:00): Gateway autonomously induces `quiet-hours` → No regulation required.
- Refrain completely from designating `every < 10m` irrespective of Creator engagement — urgent dispatches demand webhooks.
- Sustained dormancies manifesting after 20:00 escalate towards `every = "12h"`, eliminating wasteful overnight token depletion.
- `HEARTBEAT.md` is STRICTLY limited to < 20 lines.

**Adaptive Recalibration (via `gateway.config.patch`):**
```
Creator signs on → gateway.config.patch(heartbeat.every = "30m")
Creator remains inactive for 1hr → gateway.config.patch(heartbeat.every = "3h")
Post-20:00 void of tasks → gateway.config.patch(heartbeat.every = "12h")
```

**Troubleshooting Undelivered Heartbeats:**
- `quiet-hours` → Legitimate suspension outside activeHours.
- `requests-in-flight` → The primary session is preoccupied (will auto-resolve).
- `empty-heartbeat-file` → `HEARTBEAT.md` requires populated content.
- `alerts-disabled` → Validate the associated channel architecture.

---

## 6. Nightly Tasks (3 AM – via OpenClaw native cron job)

The system relies natively on **OpenClaw cron jobs** (`0 3 * * *`).
The Gateway dispatches a system event to the main session → The Agent subsequently calls `nightly_indexer.py`.

**Nightly Directives:**
- Sweeps lifecycle tiers: ACTIVE → DECAYING → ARCHIVED.
- Refreshes computational Salience: `log(access_count + 1)` applied systematically.
- Executes automated consolidation: When clustered DECAYING nodes > 20 → Constructs an overarching `[INSIGHT]` summary.
  - Retrospectively aims `CAUSED_BY` at obsolete nodes.
  - Implants a redirect within `archived_alias.json` (leaving the native edge undisturbed).
  - Archives obsolete nodes sequentially to `/memory/archive/YYYY-MM/`.
- Rejuvenates the `causal_index.json` structure.
- Obliterates ARCHIVED nodes from ChromaDB.
- Recalibrates `_INTEGRITY` hashes following substantial architecture changes.
- Issues alerts if execution spills past 03:30 → Signifying critical job delays.

**Emergency Manual Trigger:**
```bash
python3 /memory/4_OPERATIONAL_DATA/rag/nightly_indexer.py
```

---

## 7. Weekly Tasks (Saturdays 2 AM – via OpenClaw native cron job)

Job execution `0 2 * * 6` transmits the system event → Agent invokes `weekly_graph_validator.py`.

**Weekly Directives:**
- Cycle interdiction (Detects looping logic: A → B → C → A).
- Audits and prunes dangling edges.
- Validates the viability of archive references.
- Validates structural integrity within `archived_alias.json`.
- Monitors node explosions (Alert mechanisms trip if active nodes > 5000).
- Harmonizes logical consistency between `causal_index.json` and Markdown tags.
- Scribes a comprehensive report into `REFLECTION_CYCLES.md`.
- Acute infractions will definitively trigger Reflection Mode coupled with alert dispatches to the Creator.

**Manual Invocation:**
```bash
python3 /memory/4_OPERATIONAL_DATA/rag/weekly_graph_validator.py
```

---

## 8. Parity Audit (Sundays 2 AM – via OpenClaw cron job)

Cron job `0 2 * * 0` dispatches the system event → Agent executes the Parity Audit.

**Audit Directives:**
- **Scan:** Itemizes all tools/scripts defined inside specification documents (.md) correlating against genuine implementation files.
- **Compare:** Each tool annotated within `TOOL_HARNESS_LAYER.md` → Verifies the corresponding file's physical existence.
- **Classify Gaps:**

| Severity | Definition | Resolution Plan |
|----------|-----------|----------|
| `LOW` | Declared but unimplemented (Backlogged) | Register onto report, structure next sprint |
| `MEDIUM` | Structural/Schema discrepancy | Prioritize nightly fixes |
| `CRITICAL` | Implemented without declaration / Severe registry evasion | Alert Creator + forcibly initiate Reflection Mode |

- **Output Generation:** Dispatches a structured gap report directly to `memory/PARITY_AUDIT.md` (append-only methodology).
- Cumulates gaps → If CRITICAL > 0 → Broadcasts alerts to the Creator.

**Manual Invocation:**
```bash
# Execute comprehensive audit (post-upgrade routine / suspicion of systemic gaps)
python3 /memory/4_OPERATIONAL_DATA/rag/parity_audit.py --full
```

---

## 9. Cron Job Registry

Following `cron_setup.py` execution, curate and actively manage `5_WORKING_STATE/CRON_REGISTRY.md`:

```markdown
# Cron Job Registry — [Agent Name/ID]
Updated: YYYY-MM-DD

| Job ID | Job Designator | Execution Schedule | Type | Delivery Mode | Present Status |
|---|---|---|---|---|---|
| abc123 | Nightly Memory Indexer | 0 3 * * * Asia/HCM | systemEvent | none | ✅ active |
| def456 | Weekly Graph Validator | 0 2 * * 6 Asia/HCM | systemEvent | none | ✅ active |
| ghi789 | Monday Health Check    | 0 9 * * 1 Asia/HCM | systemEvent | announce | ✅ active |
| jkl012 | Parity Audit           | 0 2 * * 0 Asia/HCM | systemEvent | none | ✅ active |
```

**Management Protocol:**
```bash
openclaw cron list --agent <id>     # Retrieve actual ID dynamically
```

---

## 10. Structural Shadow Evaluator

**The Deterministic Core (Code-Level Application — `shadow_validator.py`):**

**Input Schema Design:**
```yaml
shadow_input:
  action: "write" | "read" | "expand"      # Operational classification
  type: "fact" | "insight" | ...             # Memory syntax (write operations exclusively)
  keyword: "<string>"                        # Short keyword (write operations exclusively)
  node_id: "<string>"                        # Target node (read/expand operations exclusively)
  derived_confidence: 0.00–1.00              # Prescriptive confidence values
  edge_direction: "source" | "target"        # Vectors of edge mutation
```

**Output Schema Design:**
```yaml
shadow_output:
  status: "PASS" | "BLOCK" | "WARN"
  reason: "<human-readable explanation>"
  blocked_by: "type_missing" | "keyword_invalid" | "confidence_low" | "direction_violation" | null
  repair_hint: "<prompt_repair_instructions>" | null     # Available only upon BLOCKS
```

**Validation Directives:**
- Pre-write inspection: Are TYPE and KEYWORD structurally legitimate?
- Key length ≤40 chars, devoid of special symbols (excluding core spaces)?
- Variance risks, processing latency, financial overheads, boundary compliance?
- Derived Confidence depreciates < 0.65 → SYSTEM BLOCK.
- Directional safety subversion → SYSTEM BLOCK.
- PINNED/IMMUTABLE segments relegate exclusively as **targets** (never a source point of mutation).
- Structural failures log "Shadow deterministic fail – blocked" + dispense `repair_hint`.

**Prompt Foundation:** Elucidates rationalizations only posterior to successful deterministic PASS gates. Never overrides computational determination.

**Structured Output Retries:**
- When the LLM broadcasts a misaligned output structure (invalid JSON, omitted keys) → A prompt repair cycle initiates.
- Hard limit of 2 repair cycles → Sustained failure → Extracts salvageable partial data + SYSTEM BLOCK.
- Retry metrics commit sequentially to the localized field `STRUCTURED_RETRY_COUNT`.

---

## 11. Session Intelligence

Routine measurement of dynamic Token economics integrated uniquely into every session:

```yaml
session:
  max_budget_tokens: 150000       # Ceiled boundary defining every session module
  compact_after_turns: 12          # Consolidates transcript subsequent to 'N' defined turns
  structured_retry_limit: 2        # Peak threshold governing repair events
```

**Volumetric Budget Tracking:**
- Continual evaluation per communicative turn regarding consumed metrics (Input & Output streams).
- `budget_percent > 85%` → Re-aligns into restrictive mode parameters:
  - Compresses RAG scale outputs (max_nodes shrinks to 3).
  - Shortens localized data context (Restricts inputs strictly to top-sequence chunks).
  - Flags the Creator structurally: "Computational budgetary limits sit at 85%; restrictive mode established."
- `budget_percent > 95%` → Projects comprehensive critical alarms urging rapid wrap-up parameters.

**Transcript Compaction Procedures:**
- Activates specifically when `turns > compact_after_turns`:
  - Enshrines 'N' recent turn variations intact.
  - Summarizes prior localized events into brief data fragments.
  - Offloads sequence mapping strictly onto `SESSION_TOKENS` dimensional mapping.

---

## 12. Core Evaluative Checklist (Prerequisite to Every Resolution)

1. Examine specific modal mandates delegated uniquely by the Creator?
   → If detected → Transition rapidly into Minimal / Core-only / Reflection modes.
2. Initialize structural Runtime Mode (Standard operating default) or structurally required Reflection variations.
3. **Validating Tool Harness parameters:** Has Tool Registry properly loaded? Are functional Authorization measures structurally cleared?
4. Process through Shadow Evaluator gates (Engaging deterministic evaluations pre-write).
5. Post execution of written protocols → Enact Tool Harness parameters → run `write_helper` mechanisms → force re-index execution → configure nightly evaluation triggers.
6. Recurring identical signals or high localized frictions surface? → Broadcast triggers signaling mandatory Reflection initiation.
7. Monitor operational heartbeat parameters dynamically: Creator status identified?
   → Configure modes appropriately (BUSY [10m] / NORMAL [30m] / IDLE [3h–12h]).
8. Analyze OpenClaw native configurations: Refer specifically to `CRON_REGISTRY.md` — Verification check of structural deployments. Valid 4-tier processing? (inclusive of localized Audit models).
9. **Budget analytical tracking:** Calculate consumed matrices per dimensional metric. Reaching an 85% capacity? → Execute economy configuration measures comprehensively.

---

## The Paramount Rulesets Establishing Structural Integrations

- The solitary structural foundation of truth rests firmly embedded within Markdown files + the deterministic generation from `write_helper.py`.
- RAG variables, `causal_index.json`, `archived_alias.json`, and isolated ChromaDB environments are distinctly designated as derived temporary caches.
- Structural Semantic summaries facilitate sequential cause-links while vastly limiting the threat posed by expanding nodal sequences.
- Major topological modifications persist exclusively nested under stringent Reflection boundaries + the Creator’s confirming oversight rules.
- Operational down-time deriving from localized failures forces the core framework directly to Minimal/Core-only protective matrices.
- OpenClaw native configurations replace historically isolated Unix crontab instances entirely without disruption.
- Comprehensive Tool Calls process fundamentally over a regulated Tool Harness format, injecting permission + detailed audit sequences directly into logical events.

---

*Arc - Builder | FEOFALLS Team | v1.9 | 2026-04-01*  
*Applicable to all Solo Agents within the FEOFALLS ecosystem*  
*ATLAS agents: Consult exclusively `ATLAS_Architecture_v1.0.md`*
