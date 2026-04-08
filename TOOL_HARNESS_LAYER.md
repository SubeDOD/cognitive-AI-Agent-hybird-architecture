# TOOL_HARNESS_LAYER.md
**Cross-cutting Tool Management Layer — Supply Chain Hardened + Write Protocol Automation**  
**Applies to:** All Solo Agents within the FEOFALLS ecosystem  
**Author:** Arc - Builder (FEOFALLS Team)  
**Date Updated:** 2026-04-03  

---

## 1. Purpose

Standardize all tool interactions by strictly channeling them through **a single entry point** to replace scattered bash script executions.

**Core Benefits:**
- **Automated Logging:** Every tool call is automatically logged into the session metadata (`TOOL_CALLS` field).
- **Permission Governance:** Prohibits agents from executing restricted tools (e.g., rewriting the Constitution).
- **Financial Tracking:** Calculates precise tool utilization counts per session.
- **Resilient Execution:** Wrappers inherently intercept failures, engage retry loops, and initiate smooth fallbacks.
- **Integrity Verification:** Authenticates the SHA-256 hash of a designated script prior to runtime.
- **Dependency Auditing:** Declares and independently verifies the exact dependency chains tied to each tool.
- **Hard Enforcement:** Utterly blocks the execution of unregistered tools (Zero passthrough).

---

## 2. Tool Registry Configuration

Formally map every executable tool authorized for the agent.

**Format:** `tool_registry.json` (Residing at `memory/4_OPERATIONAL_DATA/rag/`)

```json
{
  "version": "1.9",
  "strict_mode": true,
  "tools": [
    {
      "name": "poan_writer",
      "description": "Automated Write Protocol — generates NODE_ID + injects metadata + initiates reindex universally",
      "script": "skills/writer/poan_writer.py",
      "sha256": "PENDING_HASH",
      "permission": "any_agent",
      "isolation": "native",
      "dependencies": [],
      "input_schema": {
        "type": "string (optional) — memory type, default: auto",
        "keyword": "string (optional) — auto-detect from filename if vacant",
        "file": "string — 1 explicit file path",
        "batch": "string — directory path (executes across all localized .md files)",
        "tags": "string (optional) — comma-separated array",
        "dry-run": "bool (optional) — diagnostic preview"
      },
      "output": "structured report (injected/skipped/error per file)"
    },
    {
      "name": "poan_crawler",
      "description": "Zero-dependency web crawler — converts HTML directly to Markdown",
      "script": "skills/crawler/poan_crawler.py",
      "sha256": "PENDING_HASH",
      "permission": "any_agent",
      "isolation": "native",
      "dependencies": [],
      "input_schema": {
        "url": "string (required) — URL assigned for crawling",
        "output": "string (optional) — output file path destination",
        "markdown": "bool (optional) — confines response strictly to raw markdown"
      },
      "output": "JSON containing markdown body + metadata + explicit content hash"
    },
    {
      "name": "write_helper",
      "description": "Deterministic NODE_ID generation from TYPE + KEYWORD resulting in file commitment",
      "script": "memory/4_OPERATIONAL_DATA/rag/write_helper.py",
      "sha256": "a1b2c3d4e5f6...signing_hash_here",
      "permission": "any_agent",
      "isolation": "native",
      "dependencies": [],
      "input_schema": {
        "type": "string (required) — structural memory prefix",
        "keyword": "string (required) — string limited to 5 words, devoid of special symbols",
        "content": "string (optional) — body snippet required for hash construction"
      },
      "output": "Calculated NODE_ID string"
    },
    {
      "name": "reindex_file",
      "description": "Rapid hot re-index initiated subsequently to a successful file write",
      "script": "memory/4_OPERATIONAL_DATA/rag/reindex_file.sh",
      "sha256": "b2c3d4e5f6g7...signing_hash_here",
      "permission": "any_agent",
      "isolation": "native",
      "dependencies": [],
      "input_schema": {
        "file_path": "string (required) — path routing to target file"
      },
      "output": "Standard exit code 0/1"
    },
    {
      "name": "query",
      "description": "Cognitive Retrieval Pipeline — coordinates search, data expansion, and confidence propagation loops",
      "script": "memory/4_OPERATIONAL_DATA/rag/query.py",
      "sha256": "c3d4e5f6g7h8...signing_hash_here",
      "permission": "any_agent",
      "isolation": "native",
      "dependencies": [],
      "input_schema": {
        "query": "string (required) — designated search structure",
        "deep_reasoning": "bool (optional) — activates Deep Dive mode configurations"
      },
      "output": "ranked structural chunks + correlated confidence scores"
    },
    {
      "name": "nightly_indexer",
      "description": "Lifecycle sweeps, computational salience updates, node consolidation, and systematic archival operations",
      "script": "memory/4_OPERATIONAL_DATA/rag/nightly_indexer.py",
      "sha256": "d4e5f6g7h8i9...signing_hash_here",
      "permission": "system_only",
      "isolation": "native",
      "dependencies": [],
      "input_schema": {
        "flags": "string (optional) — --update-salience --check-consolidation --migrate"
      },
      "output": "execution report text"
    },
    {
      "name": "weekly_graph_validator",
      "description": "Cycle detection mapping, neutralization of dangling edges, alias integrity checks, and nodal explosion tracking",
      "script": "memory/4_OPERATIONAL_DATA/rag/weekly_graph_validator.py",
      "sha256": "e5f6g7h8i9j0...signing_hash_here",
      "permission": "system_only",
      "isolation": "native",
      "dependencies": [],
      "input_schema": {},
      "output": "graph status health report"
    },
    {
      "name": "parity_audit",
      "description": "Cross-reference specification documentation with hardcoded physical implementation files + dependency evaluations",
      "script": "memory/4_OPERATIONAL_DATA/rag/parity_audit.py",
      "sha256": "f6g7h8i9j0k1...signing_hash_here",
      "permission": "system_only",
      "isolation": "native",
      "dependencies": [],
      "input_schema": {
        "mode": "string (optional) — --full | --quick | --deps-only"
      },
      "output": "explicit gap report evaluating LOW/MEDIUM/CRITICAL discrepancies"
    },
    {
      "name": "compact_session",
      "description": "Transcription compaction deployed when session limits surpass defined thresholds",
      "script": "internal",
      "sha256": "INTERNAL_NO_HASH",
      "permission": "system_only",
      "isolation": "native",
      "dependencies": [],
      "input_schema": {
        "keep_recent_turns": "int (required) — quantity of localized turns shielded from summary condensation"
      },
      "output": "compacted structured transcript"
    },
    {
      "name": "constitution_edit",
      "description": "Modify Core Layer 0 parameters (Constitution) — MANDATES ABSOLUTE CREATOR CONFIRMATION",
      "script": "manual",
      "sha256": "MANUAL_NO_HASH",
      "permission": "creator_only",
      "isolation": "native",
      "dependencies": [],
      "input_schema": {
        "file": "string (required) — target document seated within 0_CONSTITUTION/",
        "change_description": "string (required) — defined parameters delineating modification scope"
      },
      "output": "Creator authorization status"
    }
  ]
}
```

### 2.1 Essential Sub-Fields

| Field | Type | Mandatory | Description |
|-------|------|-----------|-------------|
| `sha256` | string | ✅ | The calculated SHA-256 hash identifying the isolated script file. Verification occurs forcefully prior to execution. Utilize `INTERNAL_NO_HASH` or `MANUAL_NO_HASH` for internal/manual bypass states. |
| `dependencies` | array | ✅ | Definitive lists of required pip/npm packages. Exact versions must remain locked. `[]` equates to zero external integration (optimal security). |
| `isolation` | string | ✅ | `native` = local host execution, `docker` = constrained container runtime, `sandbox` = deeply restricted isolation boundary. |
| `strict_mode` | bool | ✅ (Root) | `true` = Exclusively blocks unregistered tools. `false` = Authorized passthrough (utilized specifically during architecture migrations). |

### 2.2 Dependency Declaration Format

When a specific tool invokes an external dependency array:

```json
{
  "name": "crawl4ai_wrapper",
  "script": "tools/crawl4ai_wrapper.py",
  "sha256": "abc123...",
  "permission": "any_agent",
  "isolation": "docker",
  "dependencies": [
    {"package": "crawl4ai", "version": "==0.8.6", "registry": "pypi", "sha256": "xyz789..."},
    {"package": "playwright", "version": "==1.48.0", "registry": "pypi", "sha256": "def456..."}
  ]
}
```

> **Dependency Defense Protocols:**
> 1. All invoked dependencies MUST utilize pinned versions (`==`, avoiding fuzzy parameters like `>=` or `~=`).
> 2. All invoked dependencies MUST possess an established `sha256` cryptographic hash.
> 3. Introducing a newly generated dependency MANDATES explicit Creator authorization.
> 4. Tools possessing `dependencies: []` naturally assume higher operational priority (Zero structural attack variance).
> 5. Tools wielding dependencies MUST execute specifically atop `isolation: "docker"` or `"sandbox"` parameters — `native` operation is strictly barred.

---

## 3. Permission Matrix

| Permission Level | Operational Authority | Tool Application | Caveats |
|------------------|-----------------------|------------------|---------|
| `any_agent` | Primary operating agent | write_helper, reindex_file, query | Unrestricted provided the Shadow Evaluator signifies approval. |
| `system_only` | Subsystem Orchestrator / Automated Cron routines | nightly_indexer, weekly_graph_validator, parity_audit | Automatically triggers **HARD BLOCK** mechanisms if the caller deviates from established systemic cron parameters. |
| `creator_only` | Validated human Creator explicitly verifying steps | constitution_edit | The executing agent drafts the modification scenario → Formally awaits Creator's final binding approval before proceeding. |

**System Enforcement Parameters:**
- Interfacing with a `system_only` module from outside Cron channels → Dispatches a **BLOCK** signal + archives `[SECURITY: unauthorized system_only call]`.
- Attempting usage of `creator_only` features unilaterally → Dispatches a **BLOCK** signal + formulates a propositional dispatch directed to the Creator.
- Encountering tools absent from current registry mapping → Dispatches a **BLOCK** signal + archives `[SECURITY: unregistered tool blocked]` + Triggers acute external alerts.
- Validating a flawed `SHA-256` parameter → Dispatches a **BLOCK** signal + archives `[SECURITY: integrity check failed]` + Engages lockdown protocols.

---

## 4. Execution Wrapper Architecture

**Internal structural logic driving `harness_execute()`:**

```python
import hashlib

def harness_execute(tool_name: str, caller: str, params: dict) -> Result:
    # ═══════════════════════════════════════════
    # PHASE 1: GATE — Verification Checks prior to sequence initialization
    # ═══════════════════════════════════════════
    
    # 1. Verification of Registry parameters — HARD BLOCK mechanism
    tool = registry.get(tool_name)
    if tool is None:
        log_security(f"UNREGISTERED TOOL BLOCKED: {tool_name}")
        alert_creator(f"Agent attempted to run unregistered tool: {tool_name}")
        return Result(status="BLOCKED", reason="Tool not in registry")
    
    # 2. Scrutiny of systemic Permissions — HARD BLOCK mechanism
    if tool.permission == "creator_only" and caller != "creator":
        return Result(status="BLOCKED", reason="Requires Creator approval")
    
    if tool.permission == "system_only" and caller not in ["system", "cron"]:
        log_security(f"UNAUTHORIZED system_only call: {tool_name} by {caller}")
        return Result(status="BLOCKED", reason="system_only tool called by non-system")
    
    # 3. Calculation of Integrity Parameters — SHA-256 verification
    if tool.sha256 not in ["INTERNAL_NO_HASH", "MANUAL_NO_HASH"]:
        actual_hash = sha256_file(tool.script)
        if actual_hash != tool.sha256:
            log_security(f"INTEGRITY FAILED: {tool_name} expected {tool.sha256[:16]}... got {actual_hash[:16]}...")
            alert_creator(f"Script tampered: {tool.script}")
            return Result(status="BLOCKED", reason="Integrity check failed")
    
    # ═══════════════════════════════════════════
    # PHASE 2: EXECUTION SEQUENCE — Cleared operational routing
    # ═══════════════════════════════════════════
    
    start_time = now()
    try:
        if tool.isolation == "docker":
            result = execute_in_container(tool.script, params)
        elif tool.isolation == "sandbox":
            result = execute_in_sandbox(tool.script, params)
        else:  # native operational pathway
            result = execute(tool.script, params)
    except Exception as e:
        log_error(f"Tool {tool_name} failed: {e}")
        return Result(status="ERROR", error=str(e))
    
    # ═══════════════════════════════════════════
    # PHASE 3: METRIC LOGGING — Compilation of systematic histories
    # ═══════════════════════════════════════════
    
    duration = now() - start_time
    session_log.append({
        "tool": tool_name,
        "caller": caller,
        "params": params,
        "status": result.status,
        "duration_ms": duration,
        "integrity_verified": True,
        "isolation": tool.isolation,
        "timestamp": now()
    })
    
    session.tool_calls.append(tool_name)
    
    return result

def sha256_file(filepath: str) -> str:
    """Computes SHA-256 explicit hashes derived from file contents."""
    h = hashlib.sha256()
    with open(filepath, "rb") as f:
        for chunk in iter(lambda: f.read(8192), b""):
            h.update(chunk)
    return h.hexdigest()
```

---

## 5. Isolation Modes

| Modal Designation | Functional Outline | Integration Scenarios |
|-------------------|--------------------|-----------------------|
| `native` | Integrates directly upon the native host filesystem constraints. | Employed solely for tools explicitly lacking external dependency components (`dependencies: []`). |
| `docker` | Traps execution firmly within isolated Docker containment limits. | Mandatory for operations requiring external package integrations (e.g., crawl4ai, playwright). |
| `sandbox` | Establishes restricted operational testing conditions (no outward networking, writing confined solely to output). | Compulsory for newly formulated tools awaiting definitive Creator clearance. |

> **Operational Isolation Parameters:**
> - `dependencies: []` + `isolation: "native"` → ✅ Operates flawlessly.
> - `dependencies: [...]` + `isolation: "native"` → ❌ **INITIATES SYSTEM BLOCK** — Definite Supply Chain vulnerability detected.
> - `dependencies: [...]` + `isolation: "docker"` → ✅ Operates flawlessly (File access fundamentally segregated).
> - Newly structured tools evading exhaustive audits → Compulsorily mandate `isolation: "sandbox"` requirements.

---

## 6. Session Tool Formatting Matrix

Closing session sequences automatically assemble systemic histories targeting dimensional logging protocols:

```yaml
session_tool_summary:
  total_calls: 5
  tools_used: ["write_helper", "reindex_file", "query"]
  blocked_calls: 1
  integrity_checks_passed: 4
  integrity_checks_failed: 0
  warnings: 0
  total_duration_ms: 3200
```

Metadata compiles directly into the defined `TOOL_CALLS` location standardizing memory structures.

---

## 7. Crucial Security Mandates

- Cross-cutting operations remain categorically separate extending no impact upon defined Complexity Budgets.
- The established Registry acts firmly as a structured definition list—Nexus/agent frameworks assume responsibility executing practical runtime applications.
- Subverting permission parameters directly generates a comprehensive **SYSTEM BLOCK** avoiding weak warnings completely.
- Compatibility concessions compromising baseline security features are permanently terminated by explicit design.
- The foundational execution of non-native elements absolutely forces integration inside localized Sandbox or Docker environments—native exposure runs zero tolerance levels.
- The absolute calculation of exact `SHA-256` hashes upon execution ranks non-negotiable—maliciously subverted parameters initiate an overarching system freeze immediately.
- Expansion of integrated system packages necessitates comprehensive Creator Authorization before activation overrides apply.

---

*Arc - Builder | FEOFALLS Team*  
*Supply Chain Hardened — Cross-cutting architecture layer*
