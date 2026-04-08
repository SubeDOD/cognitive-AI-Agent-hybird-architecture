# AGENT_OPERATIONAL_GUIDE.md – v1.9
**(Tương thích hoàn toàn với ARCHITECTURE_HYBRID_v1.9 & LOG_SCHEMA_v1.9)**
**Áp dụng cho:** Tất cả Solo Agent trong hệ sinh thái FEOFALLS
**Phiên bản:** v1.9
**Tác giả:** Arc - Builder (FEOFALLS Team)
**Ngày cập nhật:** 2026-04-01
**Cơ sở:** v1.723 | Nâng cấp với ATLAS v1.0 patterns (Tool Harness, Parity Audit, Session Intelligence)
**Mục tiêu:** Hướng dẫn vận hành hàng ngày — bootstrap, write/read rule, tool harness, reflection trigger, nightly/weekly tasks, heartbeat, cron management, parity audit, session intelligence, safety net, checklist — đảm bảo agent luôn deterministic, an toàn, token-efficient, chống node explosion và phù hợp mọi vai trò trong hệ sinh thái FEOFALLS.

---

## 1. Bootstrap Sequence (Dual-Prompt + Deterministic Cognitive)

**Không load full governance mỗi session.** Sử dụng **Dual-Prompt Strategy** tối ưu token.

### A. Runtime Mode (mặc định – 90–95% các task thông thường)

Load tối thiểu, ưu tiên tốc độ và an toàn:

1. Constitution Summary + Canonical Short (≤300 tokens) – từ `0_CONSTITUTION/` (PINNED/IMMUNABLE)
2. Identity Snapshot + HEURISTICS.md (≤150 tokens) – từ `1_IDENTITY/` & `2_COGNITIVE/` (PINNED)
3. Shadow Evaluator (deterministic constraints + pre-write validation) – ≤100 tokens
4. Working State Snapshot – từ `5_WORKING_STATE/`
5. RAG health check + Cognitive Retrieval Pipeline (`query.py`) – on-demand (với alias resolve)

**Không load:**
- Full Constitution
- Narrative Trajectory
- Field Sampling Digest chi tiết
- Semantic summary nodes (chỉ load khi Reflection)
- archived_alias.json (load lazy trong query.py nếu cần)

### B. Reflection Mode (chỉ kích hoạt khi có trigger rõ ràng)

Load đầy đủ để phân tích sâu, distill và consolidate.

**Trigger conditions** (xem phần 4):
- SIGNAL lặp ≥3 lần trong 24h (từ SIGNALS.md)
- Field Sampling friction ≥3
- Derived Confidence cascade < 0.65
- Đã qua 30 ngày không Reflection Cycle
- Weekly graph validator phát hiện vấn đề nghiêm trọng
- Creator yêu cầu rõ ràng ("Run Reflection Cycle")

**Load đầy đủ:**
1. Full `0_CONSTITUTION/`
2. Full Canonical Interpretations
3. Full `1_IDENTITY/`
4. Full `2_COGNITIVE/HEURISTICS.md` + proposed heuristics từ nightly
5. Field Sampling Digest gần nhất
6. Narrative Trajectory + semantic summary nodes
7. archived_alias.json (để trace chain cũ)
8. Cognitive Pipeline với Deep Dive mode

**Sau Reflection:**
- Ghi log đầy đủ vào `REFLECTION_CYCLES.md`
- Propose heuristic mới → chờ Creator approve → distill vào HEURISTICS.md
- Nếu graph validator flag vấn đề → đề xuất fix (rebuild alias, prune dangling)

### C. Core-only / Minimal Mode (Safety Net — Emergency Fallback)

**Khi nào kích hoạt:**
- Creator yêu cầu rõ ràng: "Enter Core-only Mode"
- RAG server không phản hồi trong 10 giây
- Token usage hoặc latency vượt ngưỡng
- Shadow Evaluator block liên tục ≥3 lần
- Weekly graph validator phát hiện cycle nghiêm trọng hoặc dangling > 10%

**Load:**
1. Constitution Summary
2. Canonical Interpretations
3. Identity Snapshot
4. Shadow Evaluator (deterministic part only)

**Tắt hoàn toàn:** RAG, secondary expansion, semantic summary nodes, consolidation, archived_alias.json resolve, nightly/weekly tasks.

---

## 2. Write Rule (Deterministic & Safe – sau mọi thay đổi memory)

**Quy trình write v1.9 — thêm Tool Harness wrapper:**

1. LLM chỉ được phép viết **TYPE** và **KEYWORD** (không viết NODE_ID):
   ```
   TYPE: fact
   KEYWORD: gemini benchmark Q1 2026
   ```

2. Shadow Evaluator kiểm tra pre-write (xem phần 9 cho I/O Schema chi tiết):
   - Có TYPE và KEYWORD hợp lệ không?
   - KEYWORD ≤40 ký tự, không chứa ký tự đặc biệt ngoài dấu cách?
   - Nếu thiếu/sai → block + trả prompt sửa:
     "Thiếu TYPE hoặc KEYWORD. Hãy viết lại theo format: TYPE: fact, KEYWORD: gemini benchmark"

3. Sau khi pass Shadow → gọi qua **Tool Harness** [MỚI v1.9]:
   ```python
   # Cách mới (v1.9 — qua harness):
   harness_execute("write_helper", "agent", {"type": "fact", "keyword": "gemini benchmark Q1 2026"})
   
   # Cách cũ (v1.723 — vẫn hoạt động nhưng không tracking):
   # python3 /memory/4_OPERATIONAL_DATA/rag/write_helper.py \
   #   --type "fact" --keyword "gemini benchmark Q1 2026"
   ```
   → Script sinh NODE_ID với short-hash (collision-resistant) và ghi file.
   → Harness tự động log tool call vào session metadata (TOOL_CALLS field).

4. Sau khi file được ghi → reindex qua harness:
   ```python
   # Cách mới (v1.9):
   harness_execute("reindex_file", "agent", {"file_path": "<đường_dẫn_file>"})
   harness_execute("nightly_indexer", "system", {"flags": "--update-salience --check-consolidation"})
   
   # Cách cũ (v1.723 — vẫn hoạt động):
   # bash /memory/4_OPERATIONAL_DATA/rag/reindex_file.sh <đường_dẫn_file>
   # python3 /memory/4_OPERATIONAL_DATA/rag/nightly_indexer.py --update-salience --check-consolidation
   ```

---

## 3. Cognitive Retrieval Pipeline (query.py – v1.722 — không thay đổi)

**Pipeline hardcoded:**
1. Vector + BM25 search → lấy top-k chunks
2. Parse NODE_ID (deterministic từ write_helper.py)
3. Temporal filter + VALID_UNTIL warning
4. **Circuit Breaker – Tiered Expansion:**
   - `initial_rerank_score < 0.5` → Hard Stop
   - `Derived Confidence > 0.75` → Selective Expand (max_nodes=5, depth=2)
   - Prompt chứa `--deep-reasoning` → Deep Dive (max_nodes=8, depth=2)
   - Mặc định: không expand
5. Check archived_alias.json (redirect nếu node archived)
6. Confidence propagation
7. Salience rerank
8. Shadow Evaluator (cutoff 0.65 + direction safety)
9. Context builder → LLM

---

## 4. Reflection Trigger (Adaptive)

- SIGNAL lặp ≥3 lần trong 24h (từ SIGNALS.md)
- Field Sampling phát hiện friction lặp lại ≥3 lần
- Derived Confidence cascade < 0.65
- Đã qua 30 ngày không Reflection Cycle
- Weekly graph validator phát hiện vấn đề nghiêm trọng
- Creator yêu cầu "Run Reflection Cycle"

**Khi trigger:** Agent tự động chuyển Reflection Mode, load full governance.

---

## 5. Heartbeat & Token Economy [CẬP NHẬT v1.723]

Heartbeat chạy trên **main session** theo checklist trong `HEARTBEAT.md`. Agent tự điều chỉnh tần suất theo 3 chế độ:

| Chế độ | `every` | Điều kiện |
|---|---|---|
| **BUSY** | `10m` | Creator đang online, task cấp thiết, có notifications pending |
| **NORMAL** | `30m` | Làm việc bình thường trong `activeHours` |
| **IDLE** | `3h` → `5h` → `12h` | Hệ thống nhàn, không có task đang chờ |

**Nguyên tắc bắt buộc:**
- Ngoài `activeHours` (mặc định 08:00–22:00): Gateway tự skip (quiet-hours) → không cần điều chỉnh
- Không bao giờ đặt `every < 10m` dù Creator đang online — nếu cần nhanh hơn, dùng webhook
- Khi idle từ 20:00 trở đi → target `every = "12h"` để qua đêm không đốt token
- `HEARTBEAT.md` BẮT BUỘC giữ < 20 dòng

**Agent tự điều chỉnh (dùng gateway.config.patch):**
```
Creator bắt đầu làm việc → gateway.config.patch(heartbeat.every = "30m")
Creator idle 1h           → gateway.config.patch(heartbeat.every = "3h")
Sau 20:00, không có task  → gateway.config.patch(heartbeat.every = "12h")
```

**Troubleshooting khi heartbeat bị skip:**
- `quiet-hours` → ngoài activeHours (bình thường)
- `requests-in-flight` → main session đang bận (tự hết)
- `empty-heartbeat-file` → HEARTBEAT.md trống → thêm nội dung
- `alerts-disabled` → kiểm tra cấu hình channel

---

## 6. Nightly Tasks (3 AM – via OpenClaw cron job) [CẬP NHẬT v1.723]

Thay vì chạy OS crontab, v1.723 dùng **native OpenClaw cron job** (`0 3 * * *`).
Gateway gửi system event đến main session → agent gọi `nightly_indexer.py`.

**Nội dung nightly:**
- Lifecycle sweep: ACTIVE → DECAYING → ARCHIVED
- Salience update: `log(access_count + 1)` cho mọi node
- Automatic consolidation: Cluster DECAYING > 20 nodes → tạo [INSIGHT]-summary
  - Trỏ CAUSED_BY về node cũ
  - Ghi redirect vào archived_alias.json (không sửa edge gốc)
  - Archive node cũ vào `/memory/archive/YYYY-MM/`
- `causal_index.json` regenerate
- Prune ARCHIVED khỏi ChromaDB
- Update `_INTEGRITY` hash nếu có thay đổi lớn
- Alert nếu script chạy sau 03:30 → cron job bị delay

**Lệnh thủ công (emergency):**
```bash
python3 /memory/4_OPERATIONAL_DATA/rag/nightly_indexer.py
```

---

## 7. Weekly Tasks (thứ 7 2h sáng – via OpenClaw cron job) [CẬP NHẬT v1.723]

Cron job `0 2 * * 6` gửi system event → agent gọi `weekly_graph_validator.py`.

**Nội dung weekly:**
- Cycle detection (A → B → C → A)
- Dangling edges check
- Archive reference check
- Alias integrity check (archived_alias.json)
- Node explosion report (active nodes > 5000 → cảnh báo)
- Consistency giữa causal_index.json vs Markdown tags
- Ghi report vào REFLECTION_CYCLES.md
- Nếu vấn đề nghiêm trọng → trigger Reflection Mode + cảnh báo Creator

**Lệnh thủ công:**
```bash
python3 /memory/4_OPERATIONAL_DATA/rag/weekly_graph_validator.py
```

---

## 7b. Parity Audit (Chủ nhật 2h sáng – via OpenClaw cron job) [MỚI v1.9]

Cron job `0 2 * * 0` gửi system event → agent chạy Parity Audit.

**Nội dung audit:**
- **Scan:** Liệt kê tất cả tools/scripts trong spec (.md) và implementation thực tế (files).
- **Compare:** Mỗi tool khai báo trong TOOL_HARNESS_LAYER.md → kiểm tra file tồn tại?
- **Classify gaps:**

| Severity | Định nghĩa | Hành động |
|----------|-----------|----------|
| `LOW` | Tool khai báo nhưng chưa implement (backlog) | Log, review next sprint |
| `MEDIUM` | Schema mismatch (spec vs code khác nhau) | Flag cho nightly fix |
| `CRITICAL` | Tool đang dùng nhưng không có trong registry | Alert Creator + trigger Reflection Mode |

- **Output:** Gap report ghi vào `memory/PARITY_AUDIT.md` (append-only).
- Đếm tổng gaps → nếu CRITICAL > 0 → alert Creator.

**Lệnh thủ công:**
```bash
# Chạy full audit (sau mỗi version upgrade hoặc khi nghi ngờ gap)
python3 /memory/4_OPERATIONAL_DATA/rag/parity_audit.py --full
```

**Lưu ý:** Sau mỗi lần nâng cấp version (v1.723 → v1.9), BẮT BUỘC chạy parity audit thủ công.

---

## 8. Cron Job Registry [CẬP NHẬT v1.9]

Sau khi chạy `cron_setup.py`, tạo và duy trì `5_WORKING_STATE/CRON_REGISTRY.md`:

```markdown
# Cron Job Registry — [Agent Name/ID]
Cập nhật: YYYY-MM-DD

| Job ID | Tên | Schedule | Loại | Delivery | Status |
|---|---|---|---|---|---|
| abc123 | Nightly Memory Indexer | 0 3 * * * Asia/HCM | systemEvent | none | ✅ active |
| def456 | Weekly Graph Validator | 0 2 * * 6 Asia/HCM | systemEvent | none | ✅ active |
| ghi789 | Monday Health Check    | 0 9 * * 1 Asia/HCM | systemEvent | announce | ✅ active |
| jkl012 | **Parity Audit**       | **0 2 * * 0 Asia/HCM** | **systemEvent** | **none** | **✅ active** |
```

**Duy trì:**
```bash
openclaw cron list --agent <id>     # Lấy ID thực tế sau khi tạo
```

---

## 9. Shadow Evaluator thực thi [FORMALIZED v1.9]

**Phần deterministic (code-level — shadow_validator.py):**

**Input Schema (v1.9):**
```yaml
shadow_input:
  action: "write" | "read" | "expand"      # loại thao tác
  type: "fact" | "insight" | ...             # memory type (chỉ khi write)
  keyword: "<string>"                        # keyword (chỉ khi write)
  node_id: "<string>"                        # node đích (chỉ khi read/expand)
  derived_confidence: 0.00–1.00              # confidence hiện tại
  edge_direction: "source" | "target"        # hướng edge mutation
```

**Output Schema (v1.9):**
```yaml
shadow_output:
  status: "PASS" | "BLOCK" | "WARN"
  reason: "<human-readable explanation>"
  blocked_by: "type_missing" | "keyword_invalid" | "confidence_low" | "direction_violation" | null
  repair_hint: "<prompt sửa lỗi>" | null     # chỉ khi BLOCK
```

**Validation rules (không thay đổi từ v1.722):**
- Pre-write validation: TYPE + KEYWORD có hợp lệ không?
- KEYWORD ≤40 ký tự, không chứa ký tự đặc biệt ngoài dấu cách?
- Risk variance, delay, cost, boundary check
- Derived Confidence < 0.65 → BLOCK
- Edge direction safety violation → BLOCK
- PINNED/IMMUTABLE nodes chỉ được là **target** (không phải source) của edge mutation
- Nếu BLOCK → log "Shadow deterministic fail – blocked" + return repair_hint

**Phần prompt-based:** Chỉ giải thích sau khi deterministic PASS. Không override deterministic result.

**Structured Output Retry [MỚI v1.9]:**
- Khi LLM trả output sai format (JSON invalid, thiếu field) → gửi repair prompt
- Max 2 repair attempts → nếu vẫn fail → extract partial data + BLOCK
- Retry count ghi vào metadata field `STRUCTURED_RETRY_COUNT`

---

## 10. Session Intelligence [MỚI v1.9]

Agent tracking token usage mỗi session:

```yaml
session:
  max_budget_tokens: 150000       # budget tối đa mỗi session
  compact_after_turns: 12          # compact transcript sau N turns
  structured_retry_limit: 2        # max retry khi output sai format
```

**Budget tracking:**
- Mỗi turn, ước lượng tokens đã dùng (input + output)
- `budget_percent > 85%` → chuyển mode tiết kiệm:
  - Giảm RAG expansion (max_nodes=3 thay vì 5)
  - Rút gọn context (chỉ top-3 chunks)
  - Thông báo Creator: "Session budget đạt 85%, chuyển mode tiết kiệm"
- `budget_percent > 95%` → cảnh báo sắp hết budget, đề xuất wrap-up

**Transcript compaction:**
- Khi `turns > compact_after_turns` → auto-compact:
  - Giữ N turns gần nhất nguyên vẹn
  - Tóm tắt turns cũ thành summary block
  - Ghi `SESSION_TOKENS` vào metadata

---

## 11. Checklist mỗi session (kiểm tra trước khi trả lời)

1. Kiểm tra mode yêu cầu từ Creator?
   → Nếu có → chuyển Minimal / Core-only / Reflection
2. Load Runtime Mode (mặc định) hoặc Reflection Mode (nếu trigger)
3. **[MỚI v1.9] Kiểm tra Tool Harness:** Tool registry loaded? Permission matrix sẵn sàng?
4. Chạy Shadow Evaluator (pre-write nếu có write — dùng I/O Schema v1.9)
5. Sau khi viết file memory → Tool Harness → write_helper → reindex → nightly flags
6. Nếu SIGNAL lặp hoặc friction cao → flag lên lịch Reflection
7. Theo dõi heartbeat mode: creator online?
   → BUSY (10m) / NORMAL (30m) / IDLE (3h–12h)
8. Cron jobs: kiểm tra CRON_REGISTRY.md — đã setup chưa? Đủ 4 jobs (bao gồm Parity Audit)?
9. **[MỚI v1.9] Session budget:** Ước lượng tokens đã dùng. Gần 85%? → chuyển mode tiết kiệm

---

## Lưu ý quan trọng nhất v1.9

- Source-of-truth duy nhất là Markdown files + NODE_ID do write_helper.py sinh
- RAG, causal_index.json, archived_alias.json, ChromaDB chỉ là derived cache
- Semantic summary nodes giữ causal chain nhưng giảm node explosion triệt để
- Tất cả thay đổi cấu trúc chỉ được thực hiện trong Reflection Mode + Creator confirm
- RAG down hoặc nightly/weekly fail → tự động fallback Core-only Mode
- **[v1.723]** Heartbeat idle: mặc định 3h, tối đa 12h. Không bao giờ đốt token vô lý
- **[v1.723]** Cron jobs là native OpenClaw — không dùng OS crontab
- **[MỚI v1.9]** Mọi tool call nên đi qua Tool Harness → logging + permission check tự động
- **[MỚI v1.9]** Parity Audit chạy weekly → phát hiện gap giữa spec và implementation
- **[MỚI v1.9]** Shadow Evaluator có I/O Schema rõ ràng → Structured Output Retry (max 2)
- **[MỚI v1.9]** Session Intelligence → budget tracking + transcript compaction

**Cách nâng cấp từ v1.723 → v1.9:**
1. Copy đè AGENT_OPERATIONAL_GUIDE.md, ARCHITECTURE_HYBRID.md, LOG_SCHEMA.md
2. Tạo file mới: `memory/TOOL_HARNESS_LAYER.md`, `memory/PARITY_AUDIT.md`
3. Thêm cron job Parity Audit: `openclaw cron create --schedule "0 2 * * 0"` (hoặc chạy `cron_setup.py` lại)
4. Cập nhật CRON_REGISTRY.md (thêm Parity Audit row)
5. Chạy full Parity Audit thủ công: `python3 /memory/4_OPERATIONAL_DATA/rag/parity_audit.py --full`
6. Theo dõi tuần đầu: tool harness logs, parity gaps, session budget usage

---

*Arc - Builder | FEOFALLS Team | v1.9 | 2026-04-01*
*Áp dụng cho tất cả Solo Agents trong hệ sinh thái FEOFALLS*
*ATLAS agents xem: ATLAS_Architecture_v1.0.md*
