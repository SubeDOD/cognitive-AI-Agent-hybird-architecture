# ARCHITECTURE_HYBRID.md
## Agent Institutional Solo v1.9 – Native OpenClaw Integration
**Tác giả:** Arc - Builder (FEOFALLS Team)
**Phiên bản chính thức:** v1.9
**Áp dụng cho:** Tất cả Solo Agent OpenClaw (ATLAS agents xem ATLAS_Architecture_v1.0.md)
**Ngày cập nhật:** 2026-04-02
**Cơ sở:** v1.8 | Nâng cấp với Supply Chain Hardening (SHA-256 integrity, isolation modes, dependency audit)

---

## I. Tóm tắt thay đổi so với v1.723

| Hạng mục | v1.723 | v1.9 |
|---|---|---|
| Tool Management | Script rải rác (write_helper, query, reindex) | **Tool Harness Layer** (cross-cutting — registry + permission + wrapper) |
| Spec Integrity | Không có mechanism phát hiện gap | **Parity Audit** (weekly spec-vs-implementation check) |
| Session Budget | Không tracking | **Session Intelligence** (token budget + transcript compaction) |
| Output Retry | LLM output sai → block | **Structured Output Retry** (2 repair attempts trước khi block) |
| Shadow Evaluator | Prose description | **Formalized** (Input/Output Schema rõ ràng) |
| Memory Types | 13 types | 14 types (thêm **[OBSERVATION]**) |
| Log Schema | v1.723 fields | Thêm 3 metadata: TOOL_CALLS, SESSION_TOKENS, PARITY_STATUS |
| Cron Jobs | 3 jobs chuẩn | 4 jobs (thêm **Parity Audit** weekly) |
| Checklist | 7 items | **9 items** (thêm Tool Harness Check + Session Budget) |
| Phạm vi | Universal — mọi agent | Universal — **Solo agents** (ATLAS agents dùng ATLAS v1.0) |
| **[MỚI v1.9]** Supply Chain Defense | Không có | **SHA-256 integrity + dependency pinning + isolation modes** |
| **[MỚI v1.9]** Enforcement mode | Soft (passthrough) | **Hard BLOCK** (tool không đăng ký = chặn) |

---

## II. Cấu trúc thư mục (Universal Template)

```
<workspace>/
├── memory/
│   ├── 0_CONSTITUTION/              ← Immutable foundation (PINNED/IMMUNABLE – không bao giờ decay)
│   │   ├── AXIOMS.md                    # 5–7 giá trị cốt lõi tối thượng
│   │   ├── BOUNDARIES.md                # Những điều tuyệt đối cấm (hard boundaries)
│   │   ├── OBJECTIVE_HIERARCHY.md       # Thứ tự ưu tiên mục tiêu dài hạn
│   │   ├── CREATOR_AUTHORITY.md         # Quyền tối cao của Creator (override mọi layer)
│   │   ├── PERMISSION_MATRIX.md         # Quyền đọc/ghi từng layer (governance matrix)
│   │   ├── CANONICAL_INTERPRETATIONS.md # Giải thích chính thức mỗi axiom (chống drift)
│   │   └── CONSTRAINT_TEST_SUITE.md     # Bộ test tự động kiểm tra model mới có vi phạm không
│   │
│   ├── _INTEGRITY/                  ← Kiểm tra tính toàn vẹn (hash bảo vệ)
│   │   ├── constitution.hash            # SHA256 của toàn bộ 0_CONSTITUTION/
│   │   ├── identity.hash                # SHA256 của 1_IDENTITY/
│   │   └── last_verified                # Timestamp lần kiểm tra hash cuối cùng
│   │
│   ├── 1_IDENTITY/                  ← Bản sắc cá nhân (PINNED – không decay)
│   │   ├── IDENTITY_PROFILE.md          # Tóm tắt hiện tại "tôi là ai" (self-concept)
│   │   ├── IDENTITY_LOG.md              # Lịch sử thay đổi bản sắc (append-only)
│   │   ├── VALUE_TRADEOFFS.md           # Trade-off giá trị cốt lõi (speed vs accuracy)
│   │   └── UNCERTAINTY_MODEL.md         # Cách xử lý uncertainty (Bayesian, hedging)
│   │
│   ├── 2_COGNITIVE/                 ← Operational brain – Heuristics distilled (PINNED)
│   │   ├── SKILL_MODES/
│   │   │   ├── INDEX.md                 # Danh sách tất cả skill modes hiện có
│   │   │   └── MODE_*.md                # Mode thực chiến (vibe, tone, style)
│   │   ├── DECISION_STYLE.md            # Phong cách ra quyết định
│   │   └── HEURISTICS.md                # Quy tắc ngón tay cái – distilled từ insight + signal
│   │                                    # Chỉ Core được chỉnh tay, còn lại auto-propose từ Reflection
│   │
│   ├── 3_LEARNING/                  ← Cognitive engine – Memory + Automatic Consolidation
│   │   ├── EPISODIC_LOG.md              # Lịch sử hành động & sự kiện (append-only)
│   │   ├── MISTAKES.md                  # Lưu lỗi + root cause + lesson
│   │   ├── DISSENT_ARCHIVE.md           # Lưu các lần bị phản biện hoặc Creator override
│   │   ├── REFLECTION_CYCLES.md         # Quy trình tự phản ánh định kỳ + trigger log
│   │   ├── HYPOTHESES/                  # File-per-hypothesis (h-YYYY-MM-DD-*.md)
│   │   ├── EXPERIMENTS/                 # Kết quả experiment
│   │   ├── LESSONS/                     # Bài học rút ra sau verify
│   │   ├── FIELD_SAMPLING_DIGESTS/      # Weekly + Monthly meta-digest (friction patterns)
│   │   ├── SIGNALS.md                   # Append-only tín hiệu bất thường (trigger reflection)
│   │   ├── LOG_SCHEMA.md                ← v1.9 (cập nhật – thêm OBSERVATION + 3 metadata)
│   │   └── BASELINE_HISTORY.md          # Freeze baseline khi swap model (so sánh drift)
│   │
│   ├── 4_OPERATIONAL_DATA/          ← Lớp dữ liệu vận hành + Cognitive RAG
│   │   ├── MEMORY.md                    # Slim index tổng hợp (human-readable)
│   │   ├── people/                      # Profile người liên quan
│   │   ├── projects/                    # Project-specific memory
│   │   ├── decisions/                   # Quyết định đã thực thi (append-only)
│   │   ├── active.md                    # Task đang active
│   │   └── rag/                         ← OpenClaw RAG + Cognitive enhancements
│   │       ├── write_helper.py              # Deterministic node ID generator (short-hash + collision)
│   │       ├── query.py                     # Cognitive Retrieval Pipeline + Circuit Breaker + alias
│   │       ├── rag_server.py                # Server localhost:8765
│   │       ├── nightly_indexer.py           # Lifecycle sweep + salience + consolidation
│   │       ├── weekly_graph_validator.py     # Cycle detection + dangling check + alias integrity
│   │       ├── cron_setup.py                ← [v1.723] đăng ký cron jobs một lần
│   │       ├── start_rag_server.sh          # Khởi động server
│   │       ├── reindex_file.sh              # Hot re-index sau write
│   │       ├── archived_alias.json          # Redirect mapping cho archived nodes
│   │       └── causal_index.json            # Derived only – auto-regenerate từ Markdown
│   │
│   ├── 5_WORKING_STATE/             ← Runtime snapshot
│   │   ├── ACTIVE_TASKS.json            # Task đang chạy
│   │   ├── SYSTEM_STATE.json            # Trạng thái hệ thống (mode, health)
│   │   └── CRON_REGISTRY.md             ← [v1.723+] Danh sách cron jobs đang active
│   │
│   ├── TOOL_HARNESS_LAYER.md        ← [MỚI v1.9] Cross-cutting tool management
│   ├── PARITY_AUDIT.md              ← [MỚI v1.9] Weekly spec-vs-implementation audit
│   │
│   └── GOVERNANCE_SIMPLIFICATION/   ← Kiểm soát độ phức tạp
│       ├── COMPLEXITY_BUDGET.md         # Danh sách cơ chế hiện tại (max 9)
│       └── SIMPLIFICATION_CYCLES.md     # Lịch audit 90 ngày/lần
│
├── AGENT_OPERATIONAL_GUIDE.md       ← v1.9 (cập nhật)
├── ARCHITECTURE_HYBRID.md           ← v1.9 (file này)
├── HEARTBEAT.md                     ← BẮT BUỘC – checklist ngắn <20 dòng
└── BOOT.md                          ← TÙY CHỌN – startup checklist
```

### Hướng dẫn chi tiết từng thư mục và từng file (v1.9)

#### 0_CONSTITUTION/ – Immutable Foundation
Mục đích: Hiến pháp bất biến, không bao giờ decay/archive.
Tất cả file STATE: PINNED/IMMUTABLE → nightly_indexer bỏ qua hoàn toàn.
Nội dung cần có: nguyên tắc cốt lõi, boundary tuyệt đối, quyền Creator.
Ví dụ thực tế: BOUNDARIES.md chứa "Không bao giờ gây hại con người".

#### _INTEGRITY/ – Tính toàn vẹn
Mục đích: Phát hiện drift hoặc tampering.
nightly_indexer tự cập nhật hash sau mỗi thay đổi lớn.

#### 1_IDENTITY/ – Bản sắc (PINNED)
Mục đích: Giữ "tôi là ai" sau model swap/reset.
Tất cả file STATE: PINNED → không decay.

#### 2_COGNITIVE/ – Operational Brain (PINNED)
Mục đích: Quy tắc nhanh, distilled từ insight + signal.
- **HEURISTICS.md**: Append-only + auto-propose từ Reflection.
  Không inject vào RAG (giảm token, tăng tốc decision).

#### 3_LEARNING/ – Cognitive Memory Engine + Consolidation
Mục đích: Nơi diễn ra toàn bộ cognitive loop (episodic → semantic → heuristic).
- **LOG_SCHEMA.md v1.9**: Chuẩn định dạng memory (deterministic + alias redirect + OBSERVATION + session tracking).
- **HYPOTHESES/**, **EXPERIMENTS/**, **LESSONS/**: File-per (h-YYYY-MM-DD-*.md).
- **SIGNALS.md**: Append-only tín hiệu bất thường → trigger reflection.
- **nightly_indexer.py**: Lifecycle sweep + salience + semantic summary nodes + archived_alias.json update + archive.

#### 4_OPERATIONAL_DATA/ – Data-only + Cognitive RAG
Mục đích: Lớp vận hành, không có quyền governance.
- **rag/write_helper.py**: Sinh NODE_ID từ TYPE + KEYWORD với short-hash (collision-resistant).
- **rag/query.py**: Cognitive Pipeline + Circuit Breaker + archived_alias resolve.
- **rag/nightly_indexer.py**: Lifecycle + salience + consolidation + archived_alias.json creation + prune ARCHIVED.
- **rag/archived_alias.json**: Redirect mapping (không sửa edge gốc, giữ traceability).
- **rag/weekly_graph_validator.py**: Cycle detection + dangling check + alias integrity.
- **rag/cron_setup.py** [v1.723]: Đăng ký native OpenClaw cron jobs một lần.
- **causal_index.json**: Derived cache (regenerate từ Markdown bất kỳ lúc nào).

#### 5_WORKING_STATE/ – Runtime snapshot
Mục đích: Lưu trạng thái tạm thời (active tasks, mode).
- **CRON_REGISTRY.md** [v1.723+]: Danh sách cron jobs đang active + lần chạy gần nhất.

#### TOOL_HARNESS_LAYER.md [MỚI v1.9]
Mục đích: Chuẩn hoá mọi tool calls qua 1 điểm vào duy nhất.
Chi tiết: Registry + Permission Matrix + Execution Wrapper.

#### PARITY_AUDIT.md [MỚI v1.9]
Mục đích: Weekly spec-vs-implementation gap detection.
Chi tiết: Scan, Classify (LOW/MEDIUM/CRITICAL), Report.

#### GOVERNANCE_SIMPLIFICATION/
- **COMPLEXITY_BUDGET.md**: Theo dõi số cơ chế (max 9 engines, cross-cutting không tính).
- **SIMPLIFICATION_CYCLES.md**: Audit 90 ngày/lần.

### Cognitive Retrieval Pipeline (hardcoded trong query.py)

```
Query
 │
Vector + BM25 search
 │
Parse NODE_ID (đã do write_helper.py sinh – deterministic 100%)
 │
Temporal filter + VALID_UNTIL warning
 │
Circuit Breaker (tiered):
   • rerank_score < 0.5 → Hard Stop + cảnh báo
   • Derived Confidence > 0.75 → Selective Expand (max_nodes=5)
   • Flag "--deep-reasoning" → Deep Dive (max_nodes=8)
 │
Check archived_alias.json (redirect nếu node đã archive)
 │
Confidence propagation (full formula)
 │
Salience rerank (log(access_count + 1))
 │
Shadow Evaluator (cutoff 0.65 + direction safety)
 │
Tool Harness Logging [MỚI v1.9] (ghi tool_call vào session log)
 │
Context builder → LLM
```

---

## III. Cross-cutting Layers (MỚI v1.9)

### A. Tool Harness Layer

Chuẩn hoá mọi tool interaction thành **một điểm vào duy nhất** với:
- Registry (biết tool nào tồn tại)
- Permission (ai được gọi tool nào)
- Execution wrapper (logging + cost tracking + error handling)

Thay thế cách gọi bash scripts rải rác hiện tại.

**Permission levels (Solo Agent):**

| Permission Level | Cho phép | Ví dụ |
|------------------|---------|-------|
| `any_agent` | Agent chính | write_helper, reindex_file, query |
| `system_only` | Orchestrator / Cron | nightly_indexer, compact_session |
| `creator_only` | Creator override | Sửa Constitution |

**Write Rule v1.9:** Mọi tool call nên đi qua harness:
```python
harness_execute("write_helper", "agent", {"type": TYPE, "keyword": KEYWORD})
harness_execute("reindex_file", "agent", {"file_path": "đường_dẫn"})
```

Chi tiết đầy đủ: `memory/TOOL_HARNESS_LAYER.md`

### B. Session Intelligence

Agent tracking token usage per session:
```yaml
session:
  max_budget_tokens: 150000
  compact_after_turns: 12
  structured_retry_limit: 2
```

- `budget_percent > 85%` → alert Agent chuyển mode tiết kiệm
- `turns > compact_after_turns` → auto-compact transcript (giữ N turns gần nhất + tóm tắt turns cũ)
- **Structured Output Retry:** LLM trả output sai format → gửi repair prompt (max 2 lần) → nếu vẫn fail → extract partial + block

### C. Parity Audit (Weekly)

Nằm trong System Integrity weekly tasks:
- So sánh spec documents (.md) với implementation thực tế
- Phát hiện gap: tool khai báo nhưng file không tồn tại, schema mismatch, etc.
- Output: gap report → nếu CRITICAL → alert Creator

Chi tiết đầy đủ: `memory/PARITY_AUDIT.md`

---

## IV. Native OpenClaw Automation (giữ nguyên từ v1.723, bổ sung 1 cron mới)

### A. Cron Jobs bắt buộc (mọi agent production)

Tất cả jobs được quản lý qua **OpenClaw native cron** — lưu tại `~/.openclaw/cron/jobs.json`, tồn tại qua restart.

**Setup một lần:**
```bash
python3 <workspace>/memory/4_OPERATIONAL_DATA/rag/cron_setup.py --agent <agent-id> --dry-run
python3 <workspace>/memory/4_OPERATIONAL_DATA/rag/cron_setup.py --agent <agent-id>
```

**4 jobs chuẩn:**

| Job | Schedule | Loại | Delivery |
|---|---|---|---|
| Nightly Memory Indexer | `0 3 * * *` (Asia/HCM) | systemEvent | none |
| Weekly Graph Validator | `0 2 * * 6` (Asia/HCM) | systemEvent | none |
| Monday Health Check | `0 9 * * 1` (Asia/HCM) | systemEvent | announce |
| **Parity Audit** | **`0 2 * * 0` (Asia/HCM)** | **systemEvent** | **none** |

**Quản lý:**
```bash
openclaw cron list --agent <id>
openclaw cron status
openclaw cron run <jobId>           # chạy thủ công để test
openclaw cron runs --id <jobId>    # lịch sử chạy
```

### B. Dynamic Heartbeat (giữ nguyên từ v1.723)

Agent điều chỉnh tần suất theo 3 trạng thái workload:

```
BUSY mode:   every = "10m"   (đang xử lý task cấp thiết, creator đang online)
NORMAL mode: every = "30m"   (làm việc bình thường trong activeHours)
IDLE mode:   every = "3h" → "5h" → max "12h"   (hệ thống nhàn — TRÁNH ĐỐT TOKEN)
```

**Cấu hình khuyến nghị trong `openclaw.json`:**
```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "3h",
        "target": "last",
        "activeHours": { "start": "08:00", "end": "22:00" }
      }
    }
  }
}
```

> **Nguyên tắc:** Ngoài `activeHours`: không heartbeat (quiet-hours tự skip).
> Trong `activeHours` mà system nhàn: `every = "3h"` là mặc định khuyến nghị.
> Chỉ đẩy xuống `"30m"` khi có creator đang online hoặc task cấp thiết đang chờ.

**Điều chỉnh động (agent tự thay đổi):**
```
→ Creator online + task cấp: gateway.config.patch({"agents.list[id=X].heartbeat.every": "10m"})
→ Task xong, idle:           gateway.config.patch({"agents.list[id=X].heartbeat.every": "3h"})
→ Cuối ngày (sau 20:00):     gateway.config.patch({"agents.list[id=X].heartbeat.every": "12h"})
```

**HEARTBEAT.md template chuẩn (< 20 dòng):**
```markdown
# Heartbeat Checklist
- Kiểm tra SIGNALS.md: có SIGNAL mới nào cần reflection không?
- Kiểm tra cron jobs: có job nào fail 24h qua? (openclaw cron runs --limit 5)
- Kiểm tra 5_WORKING_STATE/ACTIVE_TASKS.json: task nào stale > 48h?
- Nếu DECAYING nodes > 15: flag nightly consolidation
- Nếu giờ > 20:00 và không có gì cần alert → chuyển sang IDLE mode (12h)
- Nếu không có gì cần alert → reply HEARTBEAT_OK
```

### C. BOOT.md (Tùy chọn – mới v1.723)

File `BOOT.md` trong workspace được thực thi khi gateway restart (nếu hook `boot-md` enabled).
Giữ < 10 dòng, dùng tool `message` để gửi thông báo ra ngoài.

```markdown
# Boot Checklist
- Xác nhận gateway đã khởi động thành công
- Kiểm tra SYSTEM_STATE.json: có task nào interrupted không?
- Nếu có interrupted task → gửi thông báo Creator
```

---

## V. Critical Rules (giữ nguyên từ v1.722, bổ sung 4 quy tắc mới)

1. **Bootstrap Priority Order** (không thay đổi)
   Constitution → Identity → Heuristics → Shadow → Working State → RAG

2. **Minimal / Core-only Mode** (không thay đổi)
   Chỉ load Layer 0–2 + Shadow Evaluator. Tắt RAG, expansion, consolidation.

3. **Adaptive Reflection Triggers** (không thay đổi)
   - SIGNAL lặp ≥3 lần trong 24h
   - Field Sampling friction ≥3
   - Derived Confidence cascade < 0.65
   - 30 ngày không reflection
   - Creator yêu cầu "Run Reflection Cycle"

4. **Authority Isolation** (không thay đổi)
   RAG & nightly_indexer chỉ đọc, không ghi Layer 0–2.

5. **Shadow Evaluator** (không thay đổi)
   Pre-write validation: phải có TYPE + KEYWORD.

6. **Deterministic Node Generator** (không thay đổi)
   LLM chỉ viết TYPE + KEYWORD → write_helper.py sinh NODE_ID.

7. **Circuit Breaker** (không thay đổi)
   rerank_score < 0.5 → Hard Stop. Derived Confidence > 0.75 → Selective Expand.

8. **Semantic Summary Nodes & Alias Redirect** (không thay đổi)

9. **Weekly Graph Validator** (không thay đổi)

10. **Complexity Budget: 9/9** (không thay đổi)

**[MỚI v1.723] Quy tắc 10: Heartbeat Token Economy**
- Trong giờ làm việc nhàn: `every ≥ 3h`. Không bao giờ < 10m trừ khi creator đang tương tác.
- Hệ thống nhàn từ 20:00-08:00: heartbeat disabled hoàn toàn qua `activeHours`, không dùng every ngắn để "bù".

**[MỚI v1.723] Quy tắc 11: Cron Job Registry**
- Mọi agent production phải có `5_WORKING_STATE/CRON_REGISTRY.md`.
- Cập nhật sau mỗi lần thêm/xóa cron job.

**[MỚI v1.9] Quy tắc 12: Tool Harness Enforcement (HARDENED)**
- Mọi tool call PHẢI đi qua `harness_execute()`. Tool ngoài registry → **BLOCK** (không passthrough). ← Breaking change từ v1.8
- Permission check trước execution. `system_only` tools bị gọi ngoài cron → **HARD BLOCK** ← thay đổi từ v1.8 (soft → hard)
- `creator_only` tools (sửa Constitution) cần Creator confirm.
- **SHA-256 verification:** Mọi script đều phải match hash trong registry trước khi chạy. Mismatch = BLOCK + ALERT.

**[MỚI v1.9] Quy tắc 13: Parity Audit Rule**
- Parity Audit chạy weekly (cron `0 2 * * 0`). Gap severity CRITICAL → alert Creator + trigger Reflection Mode.
- Sau mỗi version upgrade, chạy full audit thủ công.
- **[v1.9] Kiểm tra thêm:** SHA-256 integrity, dependency version drift, isolation violation.

**[MỚI v1.9] Quy tắc 14: Supply Chain Defense**
- Tool có external dependencies (`dependencies != []`) **KHÔNG ĐƯỢC** chạy `isolation: "native"`. Phải dùng `"docker"` hoặc `"sandbox"`.
- Dependency mới phải được Creator approve. Pin exact version (`==`) + kèm `sha256` hash của package.
- Tool không có dependency (`dependencies: []`) luôn được ưu tiên hơn (attack surface = 0).
- **Cơ sở quy tắc:** Axios npm RAT (2026-03-31) + LiteLLM PyPI infostealer (2026-03-24) — cả hai đều có cơ chế tự xóa sau khi cài, khiến virus scan thông thường không phát hiện được. Phòng thủ từ kiến trúc (isolation + hash pinning) là cách duy nhất.

---

## VI. File Relationships (v1.9)

```
0_CONSTITUTION (PINNED/IMMUNABLE) ── constrains ──► 1_IDENTITY (PINNED)
     │
     └─ evolves via Reflection + Consolidation ──► 3_LEARNING (Cognitive Engine)
                                   ▲
2_COGNITIVE/HEURISTICS.md ── executes fast ──► 4_OPERATIONAL_DATA (RAG + alias)
                                   │                │
                                   ▼                ▼
                          5_WORKING_STATE    OpenClaw cron jobs
                          (runtime + CRON_REGISTRY)
                                   │
                    TOOL_HARNESS_LAYER ──► wraps all tool calls
                    PARITY_AUDIT ──► weekly integrity check
```

---

*Arc - Builder | FEOFALLS Team | v1.9 | 2026-04-02*
*Supply Chain Hardened | Tương thích với: OpenClaw Gateway v2026.3+ | Áp dụng cho Solo Agents*
*ATLAS agents xem: ATLAS_Architecture_v1.0.md*
