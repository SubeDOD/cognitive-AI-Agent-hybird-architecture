# PARITY_AUDIT.md – v1.9
**Weekly Spec-vs-Implementation Gap Detection + Supply Chain Integrity**
**Áp dụng cho:** Tất cả Solo Agent trong hệ sinh thái FEOFALLS
**Phiên bản:** v1.9
**Tác giả:** Arc - Builder (FEOFALLS Team)
**Ngày tạo:** 2026-04-01
**Ngày cập nhật:** 2026-04-02
**Nguồn gốc:** v1.8 Parity Audit + Supply Chain Integrity Checks

---

## 1. Mục đích

Phát hiện **gap giữa spec documents (.md) và implementation thực tế (files/scripts)** trước khi chúng trở thành production bugs.

**Vấn đề PARITY_AUDIT giải quyết:**
- Tool được khai báo trong spec nhưng file không tồn tại
- Script tồn tại nhưng không được khai báo trong registry
- Schema metadata trong LOG_SCHEMA.md khác với code thực tế
- Cron job khai báo trong CRON_REGISTRY.md nhưng không active trong gateway
- Version mismatch (file ghi v1.8 nhưng spec đã lên v1.9)
- **[MỚI v1.9] SHA-256 hash mismatch** — script đã bị sửa đổi so với registry
- **[MỚI v1.9] Dependency drift** — package installed khác version khai báo
- **[MỚI v1.9] Isolation violation** — tool có dependency chạy native thay vì docker

---

## 2. Schedule

| Trigger | Schedule | Cách chạy |
|---------|----------|-----------|
| **Cron job tự động** | `0 2 * * 0` (Chủ nhật 2h sáng) | OpenClaw cron event → agent chạy |
| **Sau version upgrade** | Thủ công | `python3 .../parity_audit.py --full` |
| **Khi nghi ngờ gap** | Thủ công | `python3 .../parity_audit.py --quick` |

---

## 3. Quy trình Audit

### Bước 1: Scan

Liệt kê tất cả items cần kiểm tra:

**Từ Spec (source-of-truth):**
- Tools trong `TOOL_HARNESS_LAYER.md` → danh sách tool names + script paths
- Memory types trong `LOG_SCHEMA.md` → danh sách prefixes hợp lệ
- Cron jobs trong `CRON_REGISTRY.md` → danh sách job IDs + schedules
- Metadata fields trong `LOG_SCHEMA.md` → danh sách required fields
- Critical rules trong `ARCHITECTURE_HYBRID.md` → danh sách rules

**Từ Implementation (thực tế):**
- Files trong `memory/4_OPERATIONAL_DATA/rag/` → scripts tồn tại
- `tool_registry.json` → tools đã đăng ký
- `openclaw cron list` → cron jobs đang active
- Memory entries gần nhất → metadata fields thực tế

### Bước 2: Compare

So sánh từng item spec vs implementation:

```
Spec tool "write_helper" → file memory/.../write_helper.py tồn tại? → OK
Spec tool "parity_audit"  → file memory/.../parity_audit.py tồn tại? → GAP (chưa implement)
Cron "Parity Audit"       → openclaw cron list có job này?           → GAP (chưa đăng ký)
```

### Bước 3: Classify Gaps

| Severity | Định nghĩa | Hành động |
|----------|-----------|----------|
| `LOW` | Tool khai báo trong spec nhưng chưa implement (backlog item) | Log vào report, review next sprint |
| `MEDIUM` | Schema mismatch — spec nói field A nhưng code dùng field B | Flag cho nightly fix, tạo [ERROR] memory entry |
| `HIGH` | **[MỚI v1.9]** Dependency version drift hoặc isolation violation | ALERT Creator + block tool cho đến khi fix |
| `CRITICAL` | Tool đang được gọi nhưng không có trong registry / file không tồn tại / **SHA-256 mismatch** | **ALERT CREATOR** + **BLOCK TOOL** + trigger Reflection Mode |

### Bước 4: Report

Ghi kết quả vào **phần cuối file này** (append-only):

```markdown
---

### Parity Audit Report — 2026-04-01 02:00 UTC

**Phiên bản spec:** v1.8
**Agent:** [Agent Name]

| # | Item | Spec | Implementation | Severity | Status |
|---|------|------|---------------|----------|--------|
| 1 | write_helper.py | TOOL_HARNESS_LAYER.md | ✅ file tồn tại | — | OK |
| 2 | parity_audit.py | TOOL_HARNESS_LAYER.md | ❌ file chưa tồn tại | LOW | BACKLOG |
| 3 | Parity Audit cron | CRON_REGISTRY.md | ❌ chưa đăng ký | MEDIUM | PENDING |

**Tổng kết:** 1 LOW, 1 MEDIUM, 0 CRITICAL
**Hành động:** Không cần alert Creator.
```

### Bước 5: Action

- `CRITICAL > 0` → **Alert Creator ngay** (qua announce hoặc message) + trigger Reflection Mode
- `MEDIUM > 3` → Flag cho nightly review
- `LOW only` → Log và tiếp tục (review định kỳ)

---

## 4. Parity Checklist (dùng cho audit thủ công)

Khi chạy audit thủ công (sau upgrade hoặc khi nghi ngờ), kiểm tra từng mục:

- [ ] Mọi tool trong `TOOL_HARNESS_LAYER.md` → file script tồn tại?
- [ ] `tool_registry.json` → mọi entry có script path hợp lệ?
- [ ] **[v1.9] SHA-256 integrity** → `sha256sum <script>` khớp với `sha256` trong registry?
- [ ] **[v1.9] Dependencies** → `pip show <pkg>` version khớp với registry `dependencies[].version`?
- [ ] **[v1.9] Isolation rule** → tool có `dependencies != []` phải có `isolation: docker/sandbox`?
- [ ] Mọi cron job trong `CRON_REGISTRY.md` → `openclaw cron list` có tương ứng?
- [ ] `LOG_SCHEMA.md` → memory entries gần nhất có đủ required fields?
- [ ] `ARCHITECTURE_HYBRID.md` → version number nhất quán trong tất cả files?
- [ ] `AGENT_OPERATIONAL_GUIDE.md` → version number nhất quán?
- [ ] Shadow Evaluator → script `shadow_validator.py` tồn tại (nếu declared)?
- [ ] RAG server → `start_rag_server.sh` chạy được? Port 8765 listening?
- [ ] `_INTEGRITY/` → hash files up-to-date?

---

## 5. Lưu ý quan trọng

- Parity Audit là **cross-cutting layer** — không tính vào Complexity Budget 9/9
- Report là **append-only** — không xóa reports cũ (giữ audit trail)
- Sau mỗi version upgrade, **BẮT BUỘC** chạy full audit thủ công
- CRITICAL gaps phải được resolve trước khi agent tiếp tục hoạt động bình thường
- **[v1.9] SHA-256 mismatch = CRITICAL** — coi như script bị tamper cho đến khi chứng minh ngược lại
- **[v1.9] Dependency không match version = HIGH** — có thể là supply chain attack
- Parity Audit **không sửa gì** — chỉ phát hiện và báo cáo. Sửa do agent/Creator quyết định

---

## 6. Audit Reports (append-only)

*Chưa có audit report. Sẽ được append sau khi chạy audit lần đầu.*

---

*Arc - Builder | FEOFALLS Team | v1.9 | 2026-04-02*
*Supply Chain Hardened — Cross-cutting layer — không tính vào Complexity Budget*
