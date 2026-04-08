# PARITY_AUDIT.md – v1.9
**Weekly Spec-vs-Implementation Gap Detection + Supply Chain Integrity**  
**Applies to:** All Solo Agents within the FEOFALLS ecosystem  
**Version:** v1.9  
**Author:** Arc - Builder (FEOFALLS Team)  
**Date Created:** 2026-04-01  
**Last Updated:** 2026-04-02  

---

## 1. Purpose

To detect **gaps between specification documents (.md) and the actual implementation (files/scripts)** before they evolve into production bugs.

**Problems solved by PARITY_AUDIT:**
- A tool is declared in the spec, but the underlying file does not exist.
- A script exists in the directory but isn't registered in the tool registry.
- Schema metadata in `LOG_SCHEMA.md` differs from the actual code.
- A cron job is declared in `CRON_REGISTRY.md` but is not active in the gateway.
- Version mismatches across files.
- **SHA-256 hash mismatches** — a script has been tampered with or modified outside the registry.
- **Dependency drift** — installed packages differ from their declared versions.
- **Isolation violations** — a tool invoking external dependencies runs natively instead of within a Docker container.

---

## 2. Schedule

| Trigger | Schedule | Execution Method |
|---------|----------|-----------|
| **Automated Cron Job** | `0 2 * * 0` (Sunday 2:00 AM) | OpenClaw cron event → Agent execution |
| **Post-version Upgrade** | Manual | `python3 .../parity_audit.py --full` |
| **When a gap is suspected** | Manual | `python3 .../parity_audit.py --quick` |

---

## 3. Audit Procedure

### Step 1: Scan

Enumerate all items requiring verification:

**From Specifications (Source of Truth):**
- Tools defined in `TOOL_HARNESS_LAYER.md` → List of tool names + script paths.
- Memory types defined in `LOG_SCHEMA.md` → List of valid prefixes.
- Cron jobs defined in `CRON_REGISTRY.md` → List of job IDs + schedules.
- Metadata fields defined in `LOG_SCHEMA.md` → List of required fields.
- Critical rules established in `ARCHITECTURE_HYBRID.md` → List of rules.

**From Implementation (Actual state):**
- Files residing in `memory/4_OPERATIONAL_DATA/rag/` → Existing scripts.
- `tool_registry.json` → Registered tools.
- Output from `openclaw cron list` → Active cron jobs.
- Recent memory entries → Actual metadata fields being utilized.

### Step 2: Compare

Cross-reference each item between the specs and the implementation:

```
Spec specifies tool "write_helper" → Verification: Does file memory/.../write_helper.py exist? → OK
Spec specifies tool "parity_audit" → Verification: Does file memory/.../parity_audit.py exist? → GAP (not implemented)
Cron "Parity Audit"                → Verification: Is this job in `openclaw cron list`?      → GAP (not registered)
```

### Step 3: Classify Gaps

| Severity | Definition | Required Action |
|----------|-----------|----------|
| `LOW` | Tool declared in spec but not yet implemented (Backlog item) | Log into report, review in the next sprint |
| `MEDIUM` | Schema mismatch — spec requires field A but code uses field B | Flag for nightly fix, create an `[ERROR]` memory entry |
| `HIGH` | Dependency version drift or isolation violation | ALERT Creator + temporarily block tool until resolved |
| `CRITICAL` | Tool is being invoked but is missing from registry / file is missing / SHA-256 mismatch | **ALERT CREATOR** + **BLOCK TOOL** + trigger Reflection Mode |

### Step 4: Report

Log findings at the **bottom of this file** (append-only):

```markdown
---

### Parity Audit Report — 2026-04-01 02:00 UTC

**Agent:** [Agent Name]

| # | Item | Spec Origin | Actual Implementation | Severity | Status |
|---|------|------|---------------|----------|--------|
| 1 | write_helper.py | TOOL_HARNESS_LAYER.md | ✅ File exists | — | OK |
| 2 | parity_audit.py | TOOL_HARNESS_LAYER.md | ❌ File missing | LOW | BACKLOG |
| 3 | Parity Audit cron | CRON_REGISTRY.md | ❌ Not registered | MEDIUM | PENDING |

**Summary:** 1 LOW, 1 MEDIUM, 0 CRITICAL
**Action Requested:** No immediate Creator alert necessary.
```

### Step 5: Action

- `CRITICAL > 0` → **Alert Creator immediately** (via announce or message) + trigger Reflection Mode.
- `MEDIUM > 3` → Flag for nightly review.
- `LOW only` → Log and proceed (review periodically).

---

## 4. Parity Checklist (For Manual Audits)

When executing a manual audit, meticulously verify:

- [ ] Every tool listed in `TOOL_HARNESS_LAYER.md` → Does the script file exist?
- [ ] `tool_registry.json` → Does every entry point to a valid script path?
- [ ] **SHA-256 integrity** → Does `sha256sum <script>` exactly match the `sha256` hash stored in the registry?
- [ ] **Dependencies** → Does `pip show <pkg>` match the exact registry version defined in `dependencies[].version`?
- [ ] **Isolation rules** → Is any tool with `dependencies != []` strictly configured to `isolation: docker/sandbox`?
- [ ] Every cron job in `CRON_REGISTRY.md` → Does it mirror an operational task in `openclaw cron list`?
- [ ] `LOG_SCHEMA.md` → Do recent memory entries feature all required fields?
- [ ] `ARCHITECTURE_HYBRID.md` → Is the version number completely consistent across all files?
- [ ] `AGENT_OPERATIONAL_GUIDE.md` → Is the version number consistent?
- [ ] Shadow Evaluator → Does the script `shadow_validator.py` exist (if declared)?
- [ ] RAG server → Does `start_rag_server.sh` execute cleanly? Is Port 8765 listening?
- [ ] `_INTEGRITY/` → Are hash files fully up-to-date?

---

## 5. Critical Notes

- The Parity Audit operates as a **cross-cutting layer** — it does not count towards the Complexity Budget (9/9).
- Reports are strictly **append-only** — older reports must not be deleted, maintaining an immutable audit trail.
- Following every system architecture upgrade, executing a full manual audit is **MANDATORY**.
- CRITICAL gaps must be completely resolved before the agent resumes normal operations.
- **An SHA-256 mismatch equals a CRITICAL failure** — the script is considered maliciously tampered with until explicitly proven otherwise.
- **Dependency version mismatches equal a HIGH severity failure** — indicating a potential supply-chain attack.
- The Parity Audit **modifies nothing** — it solely detects and reports. Remediation is at the discretion of the Agent/Creator.

---

## 6. Audit Reports (Append-only)

*No audit reports generated yet. Reports will be appended following the first audit execution.*

---

*Arc - Builder | FEOFALLS Team | v1.9 | 2026-04-02*  
*Supply Chain Hardened — Cross-cutting layer — Exempt from Complexity Budget*
