# TOOL_HARNESS_LAYER.md – v1.901
**Cross-cutting Tool Management Layer — Supply Chain Hardened + Write Protocol Automation**
**Áp dụng cho:** Tất cả Solo Agent trong hệ sinh thái FEOFALLS
**Phiên bản:** v1.901
**Tác giả:** Arc - Builder (FEOFALLS Team)
**Ngày tạo:** 2026-04-01
**Ngày cập nhật:** 2026-04-03
**Nguồn gốc:** v1.9 + Write Protocol Automation Patch
**Cơ sở vá lỗi:** v1.9 supply chain patches + Poan protocol drift incident (2026-04-03) → tự động hóa Write Protocol

---

## 1. Mục đích

Chuẩn hoá mọi tool interaction thành **một điểm vào duy nhất** thay vì gọi bash scripts rải rác.

**Lợi ích:**
- **Logging tự động:** Mọi tool call được ghi vào session metadata (TOOL_CALLS field trong LOG_SCHEMA v1.9)
- **Permission control:** Ngăn agent gọi tool vượt quyền (ví dụ: sửa Constitution)
- **Cost tracking:** Đếm số tool calls per session → giúp Session Intelligence estimate budget
- **Error handling:** Wrapper bắt lỗi, retry, và fallback thay vì crash
- **[MỚI v1.9] Integrity verification:** Xác minh hash SHA-256 của script trước khi execute
- **[MỚI v1.9] Dependency audit:** Khai báo và kiểm tra dependency chain của mỗi tool
- **[MỚI v1.9] Hard enforcement:** Chặn hoàn toàn tool không đăng ký (không passthrough)

---

## 2. Tool Registry

Khai báo mọi tool mà agent được phép sử dụng.

**Format:** `tool_registry.json` (đặt tại `memory/4_OPERATIONAL_DATA/rag/`)

```json
{
  "version": "1.901",
  "strict_mode": true,
  "tools": [
    {
      "name": "poan_writer",
      "description": "[MỚI v1.901] Automated Write Protocol — sinh NODE_ID + inject metadata + reindex trong 1 lệnh",
      "script": "skills/writer/poan_writer.py",
      "sha256": "PENDING_HASH",
      "permission": "any_agent",
      "isolation": "native",
      "dependencies": [],
      "input_schema": {
        "type": "string (optional) — memory type, default: auto",
        "keyword": "string (optional) — auto-detect từ filename nếu bỏ trống",
        "file": "string — 1 file path",
        "batch": "string — directory path (chạy trên tất cả .md)",
        "tags": "string (optional) — comma-separated",
        "dry-run": "bool (optional) — preview không ghi"
      },
      "output": "report (injected/skipped/error per file)"
    },
    {
      "name": "poan_crawler",
      "description": "[MỚI v1.9] Zero-dependency web crawler — HTML → Markdown",
      "script": "skills/crawler/poan_crawler.py",
      "sha256": "PENDING_HASH",
      "permission": "any_agent",
      "isolation": "native",
      "dependencies": [],
      "input_schema": {
        "url": "string (required) — URL cần crawl",
        "output": "string (optional) — file path đầu ra",
        "markdown": "bool (optional) — chỉ trả markdown thuần"
      },
      "output": "JSON với markdown + metadata + content hash"
    },
    {
      "name": "write_helper",
      "description": "Sinh NODE_ID deterministic từ TYPE + KEYWORD và ghi file memory",
      "script": "memory/4_OPERATIONAL_DATA/rag/write_helper.py",
      "sha256": "a1b2c3d4e5f6...signing_hash_here",
      "permission": "any_agent",
      "isolation": "native",
      "dependencies": [],
      "input_schema": {
        "type": "string (required) — memory type prefix",
        "keyword": "string (required) — max 5 từ, không ký tự đặc biệt",
        "content": "string (optional) — snippet nội dung để hash"
      },
      "output": "NODE_ID string"
    },
    {
      "name": "reindex_file",
      "description": "Hot re-index một file sau khi write",
      "script": "memory/4_OPERATIONAL_DATA/rag/reindex_file.sh",
      "sha256": "b2c3d4e5f6g7...signing_hash_here",
      "permission": "any_agent",
      "isolation": "native",
      "dependencies": [],
      "input_schema": {
        "file_path": "string (required) — đường dẫn file cần reindex"
      },
      "output": "exit code 0/1"
    },
    {
      "name": "query",
      "description": "Cognitive Retrieval Pipeline — search, expand, confidence propagation",
      "script": "memory/4_OPERATIONAL_DATA/rag/query.py",
      "sha256": "c3d4e5f6g7h8...signing_hash_here",
      "permission": "any_agent",
      "isolation": "native",
      "dependencies": [],
      "input_schema": {
        "query": "string (required) — câu hỏi tìm kiếm",
        "deep_reasoning": "bool (optional) — kích hoạt Deep Dive mode"
      },
      "output": "ranked chunks + confidence scores"
    },
    {
      "name": "nightly_indexer",
      "description": "Lifecycle sweep, salience update, consolidation, archive",
      "script": "memory/4_OPERATIONAL_DATA/rag/nightly_indexer.py",
      "sha256": "d4e5f6g7h8i9...signing_hash_here",
      "permission": "system_only",
      "isolation": "native",
      "dependencies": [],
      "input_schema": {
        "flags": "string (optional) — --update-salience --check-consolidation --migrate"
      },
      "output": "report text"
    },
    {
      "name": "weekly_graph_validator",
      "description": "Cycle detection, dangling edges, alias integrity, node explosion check",
      "script": "memory/4_OPERATIONAL_DATA/rag/weekly_graph_validator.py",
      "sha256": "e5f6g7h8i9j0...signing_hash_here",
      "permission": "system_only",
      "isolation": "native",
      "dependencies": [],
      "input_schema": {},
      "output": "graph health report"
    },
    {
      "name": "parity_audit",
      "description": "So sánh spec documents với implementation thực tế + dependency integrity check",
      "script": "memory/4_OPERATIONAL_DATA/rag/parity_audit.py",
      "sha256": "f6g7h8i9j0k1...signing_hash_here",
      "permission": "system_only",
      "isolation": "native",
      "dependencies": [],
      "input_schema": {
        "mode": "string (optional) — --full | --quick | --deps-only"
      },
      "output": "gap report (LOW/MEDIUM/CRITICAL)"
    },
    {
      "name": "compact_session",
      "description": "Compact transcript khi vượt turn limit",
      "script": "internal",
      "sha256": "INTERNAL_NO_HASH",
      "permission": "system_only",
      "isolation": "native",
      "dependencies": [],
      "input_schema": {
        "keep_recent_turns": "int (required) — số turns gần nhất giữ nguyên"
      },
      "output": "compacted transcript"
    },
    {
      "name": "constitution_edit",
      "description": "Sửa đổi Layer 0 (Constitution) — CẦN CREATOR CONFIRM",
      "script": "manual",
      "sha256": "MANUAL_NO_HASH",
      "permission": "creator_only",
      "isolation": "native",
      "dependencies": [],
      "input_schema": {
        "file": "string (required) — file trong 0_CONSTITUTION/",
        "change_description": "string (required) — mô tả thay đổi"
      },
      "output": "Creator approval status"
    }
  ]
}
```

### 2.1 Trường mới trong v1.9

| Trường | Kiểu | Bắt buộc | Mô tả |
|--------|------|----------|-------|
| `sha256` | string | ✅ | SHA-256 hash của script file. Agent PHẢI verify trước execute. Dùng `INTERNAL_NO_HASH` cho built-in, `MANUAL_NO_HASH` cho manual tools |
| `dependencies` | array | ✅ | Danh sách pip/npm packages mà tool yêu cầu, pin version chính xác. `[]` = không có dependency bên ngoài (an toàn nhất) |
| `isolation` | string | ✅ | `native` = chạy trực tiếp trên host, `docker` = chạy trong container cách ly, `sandbox` = chạy trong sandbox restricted |
| `strict_mode` | bool | ✅ (ở root) | `true` = BLOCK tool không đăng ký. `false` = passthrough (chỉ dùng khi migrate) |

### 2.2 Dependency Declaration Format

Khi tool có external dependencies:

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

> **Quy tắc dependency v1.9:**
> 1. Mọi dependency PHẢI pin exact version (`==`, không dùng `>=` hay `~=`)
> 2. Mọi dependency PHẢI có `sha256` hash của package archive
> 3. Dependency mới PHẢI được Creator approve trước khi thêm vào registry
> 4. Tool có `dependencies: []` luôn được ưu tiên hơn tool có dependencies (attack surface = 0)
> 5. Tool có dependencies PHẢI chạy `isolation: "docker"` hoặc `"sandbox"` — KHÔNG ĐƯỢC chạy `"native"`

---

## 3. Permission Matrix

| Permission Level | Ai được gọi | Ví dụ tools | Ghi chú |
|------------------|-------------|-------------|---------|
| `any_agent` | Agent chính (trong mọi mode) | write_helper, reindex_file, query | Gọi tự do, chỉ cần Shadow Evaluator pass |
| `system_only` | Orchestrator / Cron jobs | nightly_indexer, weekly_graph_validator, parity_audit, compact_session | **[v1.9] HARD BLOCK** nếu caller không phải system/cron |
| `creator_only` | Creator trực tiếp xác nhận | constitution_edit | Agent đề xuất → Creator approve → mới execute |

**Enforcement (v1.9 — HARDENED):**
- Agent gọi tool `system_only` ngoài cron event → **BLOCK** + log `[SECURITY: unauthorized system_only call]` ← **thay đổi từ v1.8 (soft → hard)**
- Agent gọi tool `creator_only` → **BLOCK** + gửi đề xuất đến Creator
- **[MỚI v1.9] Tool không có trong registry → BLOCK** + log `[SECURITY: unregistered tool blocked]` + ALERT Creator ← **breaking change từ v1.8**
- **[MỚI v1.9] SHA-256 mismatch → BLOCK** + log `[SECURITY: integrity check failed]` + ALERT Creator

---

## 4. Execution Wrapper

**Pseudocode `harness_execute()` — v1.9 Hardened:**

```python
import hashlib

def harness_execute(tool_name: str, caller: str, params: dict) -> Result:
    # ═══════════════════════════════════════════
    # PHASE 1: GATE — Trước khi chạy bất kỳ thứ gì
    # ═══════════════════════════════════════════
    
    # 1. Registry check — HARD BLOCK (v1.9)
    tool = registry.get(tool_name)
    if tool is None:
        log_security(f"UNREGISTERED TOOL BLOCKED: {tool_name}")
        alert_creator(f"Agent attempted to run unregistered tool: {tool_name}")
        return Result(status="BLOCKED", reason="Tool not in registry")
    
    # 2. Permission check — HARD BLOCK (v1.9)
    if tool.permission == "creator_only" and caller != "creator":
        return Result(status="BLOCKED", reason="Requires Creator approval")
    
    if tool.permission == "system_only" and caller not in ["system", "cron"]:
        log_security(f"UNAUTHORIZED system_only call: {tool_name} by {caller}")
        return Result(status="BLOCKED", reason="system_only tool called by non-system")
    
    # 3. Integrity check — SHA-256 verification (v1.9 NEW)
    if tool.sha256 not in ["INTERNAL_NO_HASH", "MANUAL_NO_HASH"]:
        actual_hash = sha256_file(tool.script)
        if actual_hash != tool.sha256:
            log_security(f"INTEGRITY FAILED: {tool_name} expected {tool.sha256[:16]}... got {actual_hash[:16]}...")
            alert_creator(f"Script tampered: {tool.script}")
            return Result(status="BLOCKED", reason="Integrity check failed")
    
    # ═══════════════════════════════════════════
    # PHASE 2: EXECUTE — Chỉ khi pass tất cả gates
    # ═══════════════════════════════════════════
    
    start_time = now()
    try:
        if tool.isolation == "docker":
            result = execute_in_container(tool.script, params)
        elif tool.isolation == "sandbox":
            result = execute_in_sandbox(tool.script, params)
        else:  # native
            result = execute(tool.script, params)
    except Exception as e:
        log_error(f"Tool {tool_name} failed: {e}")
        return Result(status="ERROR", error=str(e))
    
    # ═══════════════════════════════════════════
    # PHASE 3: LOG — Ghi lại mọi thứ đã xảy ra
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
    """Tính SHA-256 hash của file."""
    h = hashlib.sha256()
    with open(filepath, "rb") as f:
        for chunk in iter(lambda: f.read(8192), b""):
            h.update(chunk)
    return h.hexdigest()
```

---

## 5. Isolation Modes (v1.9 NEW)

| Mode | Mô tả | Khi nào dùng |
|------|--------|-------------|
| `native` | Chạy trực tiếp trên host filesystem | Chỉ cho tools **không có external dependencies** (`dependencies: []`) |
| `docker` | Chạy trong Docker container riêng | Tools có external dependencies (ví dụ: crawl4ai, playwright) |
| `sandbox` | Chạy trong restricted sandbox (no network, no filesystem write ngoài output) | Tools thử nghiệm chưa được Creator fully approve |

> **Quy tắc isolation v1.9:**
> - `dependencies: []` + `isolation: "native"` → ✅ Cho phép
> - `dependencies: [...]` + `isolation: "native"` → ❌ **BLOCK** — Supply chain risk
> - `dependencies: [...]` + `isolation: "docker"` → ✅ Cho phép (cách ly filesystem)
> - Tool mới chưa qua audit → `isolation: "sandbox"` bắt buộc

---

## 6. Migration từ v1.8 → v1.9

### Breaking Changes

| Thay đổi | v1.8 | v1.9 | Ảnh hưởng |
|----------|------|------|-----------|
| Tool không đăng ký | Passthrough (chạy + warning) | **BLOCK** | Agents phải đăng ký mọi tool vào registry trước khi dùng |
| system_only enforcement | Soft (warning + chạy) | **HARD BLOCK** | Cron jobs phải declare caller = "system" |
| SHA-256 required | Không có | Bắt buộc | Mỗi script phải có hash trong registry |
| dependencies field | Không có | Bắt buộc | Khai báo `[]` nếu không có dependency |
| isolation field | Không có | Bắt buộc | Mặc định `"native"` cho tools hiện có |

### Quy trình migration

```bash
# 1. Sinh SHA-256 hash cho mọi script hiện có
cd memory/4_OPERATIONAL_DATA/rag/
for f in *.py *.sh; do
  echo "\"sha256\": \"$(sha256sum $f | cut -d' ' -f1)\"  # $f"
done

# 2. Cập nhật tool_registry.json với hash + dependencies + isolation
# 3. Set strict_mode: true
# 4. Test — chạy mỗi tool qua harness, verify không bị block sai
```

---

## 7. Session Tool Log Format

Mỗi session kết thúc, ghi summary vào metadata:

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

Dữ liệu này được ghi vào `TOOL_CALLS` field trong LOG_SCHEMA v1.9 khi tạo memory entry.

---

## 8. Lưu ý quan trọng

- Tool Harness là **cross-cutting layer** — không tính vào Complexity Budget 9/9
- Registry là **khai báo**, không phải code — Nexus/agent tự implement wrapper phù hợp runtime
- **[v1.9] Permission enforcement là HARD** — BLOCK thay vì warning (trừ `any_agent`)
- **[v1.9] Backward compatibility bị loại bỏ có chủ đích** — an toàn > tiện lợi
- **[v1.9] Tool có external dependencies PHẢI chạy trong Docker/Sandbox** — không bao giờ native
- **[v1.9] Mọi script PHẢI có SHA-256 hash** — tampered file = BLOCK ngay lập tức
- **[v1.9] Thêm dependency mới = CẦN CREATOR APPROVE** — không tự ý thêm package

---

*Arc - Builder | FEOFALLS Team | v1.9 | 2026-04-02*
*Supply Chain Hardened — Cross-cutting layer — không tính vào Complexity Budget*
