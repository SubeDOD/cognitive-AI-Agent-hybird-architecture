# LOG_SCHEMA.md – v1.9
**Cognitive Memory Block Standard – Deterministic, Safe, Collision-Resistant & Graph-Consistent**  
**Applies to:** All Solo Agents within the FEOFALLS ecosystem  

**Version:** v1.9  
**Author:** Arc - Builder (FEOFALLS Team)  
**Date Updated:** 2026-04-01  

**Primary Objectives:**
- Eliminate the risk of LLMs generating improperly formatted `NODE_ID`s entirely.
- Ensure `NODE_ID` uniqueness even with duplicate keywords and dates (via short-hash).
- Protect causal chains during consolidation (`archived_alias.json` – redirects without mutating original edges).
- Control token and context explosion via Circuit Breaker tiered expansion.
- Automate semantic summary nodes and archive redirects to prevent node explosion.
- Maintain Markdown files as the strict single source of truth.
- Track tool calls, session tokens, and parity status per memory entry.
- Maintain strict adherence to the defined Complexity Budget (≤9 modules).

---

## 1. Node ID Generation Rule (MANDATORY – Deterministic & Collision-Resistant)

**Core Principle:**
The LLM is **strictly prohibited** from generating a `NODE_ID`.
The LLM only provides 2 mandatory fields:

- **TYPE**: Placed exactly as one of the types listed in section 2.
- **KEYWORD**: A concise string (maximum 5 words, containing no special characters other than spaces).

**`write_helper.py` will automatically construct the `NODE_ID` following this format:**

```
[TYPE]-[YYYY-MM-DD]-[short-keyword][-short-hash]
```

**Short-hash Logic (Collision Avoidance):**
- Input hash = keyword + date + content_snippet[:20]
- short_hash = sha1(input)[:4] (hex format)
- If the base ID already exists → append `-short_hash`.
- If a collision persists (extremely rare) → fallback to `-nn-short_hash` (where nn = 01, 02, ...).

**Example:**

The LLM outputs:
```
TYPE: fact
KEYWORD: gemini benchmark Q1 2026
```

`write_helper.py` generates:
```
fact-2026-03-15-gemini-benchmark-q1-2026-a4f3
```

**Rationale:**
- Eradicates formatting errors entirely (100% mitigation).
- Guarantees high uniqueness (4 hex characters are sufficient for thousands of nodes daily).
- Maintains human readability.
- The Shadow Evaluator will proactively block writes omitting either TYPE or KEYWORD.

---

## 2. Memory Type Prefix (MANDATORY – Comprehensive List)

Valid prefixes (must be capitalized and encased in `[]` at the very beginning of the entry):

```
[FACT]          – Objective, verifiable data.
[INSIGHT]       – Distilled lessons derived from multiple episodes.
[ERROR]         – Documented failures paired with root cause analysis.
[DECISION]      – Executed decisions combined with their rationale.
[PREFERENCE]    – Priority values of the Creator / user.
[WORKFLOW]      – Established, repeatable processes.
[HYPOTHESIS]    – Unverified assumptions or proposed ideas.
[EXPERIMENT]    – Results from executed tests.
[LESSON]        – Extracted learnings post-verification.
[HEURISTIC]     – Fast rules of thumb, distilled into Layer 2.
[SIGNAL]        – Anomalous indicators (triggers for reflection).
[CONSTRAINT]    – Hard governance boundaries.
[ASSUMPTION]    – Presumptions requiring active monitoring.
[CRON_LOG]      – Outcomes of cron job executions (nightly/weekly/health).
[OBSERVATION]   – Raw, uninterpreted real-world feedback from the Creator or environment.
```

> **[CRON_LOG]**: Documents automation outcomes. Default STATE: ACTIVE. VALID_UNTIL: AUTO (90 days). Does not require causal relationships unless a critical error is detected.

> **[OBSERVATION]**: Documents empirical phenomena from the Creator or environment without interpretation. Example: "Creator noted the output was too verbose," or "API response time spiked by 200%." Default STATE: ACTIVE. VALID_UNTIL: AUTO (90 days). Often serves as the source of a `CAUSED_BY` for an `[INSIGHT]` or `[LESSON]` following a Reflection.

---

## 3. Metadata Block (MANDATORY – Placed immediately following the prefix)

Every entry must feature all of the following fields (prefixed with a `-` line initiator):

```markdown
- NODE_ID: fact-2026-04-01-gemini-benchmark-q1-2026-a4f3     # Generated exclusively by write_helper.py
- STATE: PINNED | IMMUTABLE | ACTIVE | DECAYING | ARCHIVED | CONSOLIDATED
- VERSION: v1 | v2 | v3 ...
- SUPERSEDES: [old NODE_ID]                                   # The node being replaced (leave blank if none)
- SUPERSEDED_BY: [new NODE_ID]                                # The node replacing this one (leave blank if none)
- TIMESTAMP: 2026-04-01                                       # Creation Date
- LAST_ACCESS: 2026-04-01                                     # Automatically updated by nightly job upon retrieval
- VALID_UNTIL: 2026-07-01 | INDEFINITE | AUTO                 # AUTO defaults to TIMESTAMP + 90 days
- CONFIDENCE: 0.00–1.00                                       # Self-assessed during creation
- SALIENCE: auto                                              # Calculated nightly: log(access_count + 1)
- SIGNAL: <optional>                                          # e.g., repeated_timeout_error
- TAGS: [tag1, tag2, tag3]                                    # Optional array
- TOOL_CALLS: []                                              # Array of tool calls executed in the session creating this entry
- SESSION_TOKENS: 0                                           # Total tokens consumed in the session creating this entry
- PARITY_STATUS: "OK" | "GAP_LOW" | "GAP_MEDIUM" | "GAP_CRITICAL"  # Outcome of the most recent parity check
- STRUCTURED_RETRY_COUNT: 0                                   # Numerical count of retries prompted by malformed outputs
```

> The metadata fields `TOOL_CALLS`, `SESSION_TOKENS`, `PARITY_STATUS`, and `STRUCTURED_RETRY_COUNT` are **optional**. If an agent hasn't implemented specific harness telemetry, it may omit or default these to empty states. These fields significantly streamline auditing and debugging.

**STATE References:**

| STATE | Description |
|---|---|
| `PINNED` | High-priority importance — immune to decay. |
| `IMMUTABLE` | Absolute foundation (exclusive to the Constitution layer). |
| `ACTIVE` | Currently operational and structurally sound. |
| `DECAYING` | Unaccessed for 60+ days — slated for archiving. |
| `ARCHIVED` | Safely migrated to `/memory/archive/YYYY-MM/`. |
| `CONSOLIDATED` | Merged into an `[INSIGHT]` summary node. |

---

## 4. Causal Relations (STRICTLY utilizes generated NODE_IDs – deterministic)

```markdown
- CAUSED_BY: hypothesis-2026-03-10-qwen-test-a1b2
- RESOLVES: error-2026-03-05-gemini-400-c3d4
- DEPENDS_ON: experiment-2026-03-10-50-tests-e5f6
- PREVENTS: constraint-2026-01-01-no-shared-keys-7890
- TRIGGERED_BY: signal-2026-03-11-repeated-timeout-1234
- IMPACTS: decision-2026-03-11-separate-api-key-abcd
- DERIVED_FROM: insight-2026-03-09-auth-conflict-pattern-ef01
```

**Direction Safety Rules (Enforced by Shadow Evaluator):**
- `PINNED`/`IMMUTABLE` nodes may exclusively act as **targets** (never sources) for edge mutations.
- `ARCHIVED` nodes: Can only be referenced via `archived_alias.json` (no direct pointers allowed).

---

## 5. Confidence Propagation Formula (Hardcoded in query.py)

```latex
DerivedConfidence_B = Confidence_A × EdgeWeight × RetrievalScore × e^(-age / τ)
```

- τ defaults to 90 days (technology insights) and 365 days (long-term facts).
- RetrievalScore: Rerank score normalized to [0,1].
- Shadow Evaluator Cutoff: A Derived Confidence < 0.65 triggers a cascade block + warning.

**Edge Weight Table (Hardcoded):**

| Relation | Weight |
|---|---|
| CAUSED_BY / RESOLVES | 0.92 |
| DEPENDS_ON / DERIVED_FROM | 0.85 |
| TRIGGERED_BY / PREVENTS | 0.78 |
| IMPACTS | 0.90 |

---

## 6. Circuit Breaker – Tiered Expansion (Hardcoded in query.py)

```python
if initial_rerank_score < 0.5:
    → Hard Stop: "Information fails minimum reliability threshold (rerank < 0.5)."
elif derived_confidence > 0.75:
    → Selective Expand: max_nodes = 5, depth = 2.
elif user_prompt contains "--deep-reasoning":
    → Deep Dive: max_nodes = 8, depth = 2.
else:
    → No Expand: Restrict to top-k chunks solely (conserves tokens).
```

---

## 7. Lifecycle, Consolidation & Archive Redirect Rules (Enforced by nightly_indexer.py)

- **PINNED / IMMUTABLE:** Fully immune (safeguarding Layers 0–2).
- **ACTIVE → DECAYING:** Triggered by 60 days of non-access.
- **DECAYING → ARCHIVED:** +30 further days of non-access, or `VALID_UNTIL` expiration.
- **ARCHIVED:** Scrubbed from ChromaDB, markdown file moved to `/memory/archive/YYYY-MM/`.

**Automatic Semantic Summary Nodes (Combats Node Explosion):**
- Weekly validation: If a cluster of `DECAYING` nodes exceeds 20 elements.
- Generates a new node: `[INSIGHT]-YYYY-MM-DD-cluster-<short-descriptor>`.
- Redirects `CAUSED_BY` to the deprecated nodes (preserving historical lineage).
- Writes an alias redirect into **`archived_alias.json`**:
  ```json
  {
    "error-2026-03-05-gemini-400": "insight-2026-03-19-auth-cluster-summary",
    "signal-2026-03-11-repeated-timeout": "insight-2026-03-19-auth-cluster-summary"
  }
  ```
- The original nodes are archived → Original edges remain unaltered.
- Only the new summary node undergoes re-indexing.

**Query Behavior (`query.py`):**
- Prior to resolving a `NODE_ID` → Considers `archived_alias.json`.
- If an alias is hit → Implements redirect + logs note: "[Redirect] Old node consolidated → utilizing summary."
- Historical traceability remains uncompromised.

---

## 8. Retrieval Constraints & Safety Limits (Hard Limits)

- Maximum Depth = 2
- Maximum Expanded Nodes Defaults = 5
- Shadow Cutoff: Derived Confidence < 0.65 → Blocks cascade + warning.
- When context expansion breaches the safety threshold → Truncation + warning: "Context expansion reached safety limit."

---

## 9. Comprehensive Entry Examples (Copy-Paste Templates)

```markdown
[FACT] Benchmark Code Gen Models Q1/2026
- NODE_ID: fact-2026-04-01-gemini-benchmark-q1-2026-a4f3
- STATE: ACTIVE
- VERSION: v1
- SUPERSEDES:
- SUPERSEDED_BY:
- TIMESTAMP: 2026-04-01
- LAST_ACCESS: 2026-04-01
- VALID_UNTIL: 2026-07-01
- CONFIDENCE: 0.95
- SALIENCE: auto
- SIGNAL:
- TAGS: [model-benchmark, code-gen, gemini]
- TOOL_CALLS: ["write_helper", "reindex_file"]
- SESSION_TOKENS: 12500
- PARITY_STATUS: OK
- STRUCTURED_RETRY_COUNT: 0

- CAUSED_BY: hypothesis-2026-03-10-qwen-vs-gemini-7b8c
- RESOLVES: decision-2026-03-11-primary-model-9d0e
- DEPENDS_ON: experiment-2026-03-10-50-tests-python-f1a2

**Content:**
Gemini v1.5 Pro attained a pass@1 rate of 92% on internal codebases, significantly outperforming Qwen 2.5 Coder (85%).
```

```markdown
[OBSERVATION] Creator expressed that the output is unnecessarily verbose
- NODE_ID: observation-2026-04-01-creator-output-too-long-c2d3
- STATE: ACTIVE
- VERSION: v1
- SUPERSEDES:
- SUPERSEDED_BY:
- TIMESTAMP: 2026-04-01
- LAST_ACCESS: 2026-04-01
- VALID_UNTIL: AUTO
- CONFIDENCE: 0.90
- SALIENCE: auto
- SIGNAL: output_length_complaint
- TAGS: [creator-feedback, output-quality]
- TOOL_CALLS: []
- SESSION_TOKENS: 8200
- PARITY_STATUS: OK
- STRUCTURED_RETRY_COUNT: 0

**Content:**
The Creator provided feedback stipulating that the agent's output is excessively long and requires condensation. This is a raw empirical observation, devoid of generated insights.
```

```markdown
[CRON_LOG] Nightly Indexer Run 2026-04-01
- NODE_ID: cron_log-2026-04-01-nightly-indexer-b2c1
- STATE: ACTIVE
- VERSION: v1
- SUPERSEDES:
- SUPERSEDED_BY:
- TIMESTAMP: 2026-04-01
- LAST_ACCESS: 2026-04-01
- VALID_UNTIL: AUTO
- CONFIDENCE: 1.0
- SALIENCE: auto
- SIGNAL:
- TAGS: [automation, nightly, cron]
- TOOL_CALLS: ["nightly_indexer"]
- SESSION_TOKENS: 0
- PARITY_STATUS: OK
- STRUCTURED_RETRY_COUNT: 0

**Results:**
- Active: 124, Decaying: 3, Archived: 0, Errors: 0
- Consolidation: Not triggered (decaying node count < 20)
- Runtime Execution: 03:02 UTC (On schedule)
```

---

## 10. Paramount Notes

- Every `NODE_ID` must exclusively originate from `write_helper.py` → Do NOT accept LLM-generated `NODE_ID`s.
- The Single Source of Truth firmly remains the Markdown files.
- `archived_alias.json` and `causal_index.json` are solely derived caches → Capable of ad-hoc regeneration.
- The Shadow Evaluator MUST block memory writes omitting a `TYPE` or `KEYWORD`.
- Overall graph consistency is robustly safeguarded by `weekly_graph_validator.py`.

---

*Arc - Builder | FEOFALLS Team | v1.9 | 2026-04-01*  
*Applicable to all Solo Agents within the FEOFALLS ecosystem*
