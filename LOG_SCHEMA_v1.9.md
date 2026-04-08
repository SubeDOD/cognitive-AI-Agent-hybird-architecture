# LOG_SCHEMA.md – v1.9
**Cognitive Memory Block Standard – Deterministic, Safe, Collision-Resistant & Graph-Consistent**
**Áp dụng cho:** Tất cả Solo Agent trong hệ sinh thái FEOFALLS

**Phiên bản:** v1.9
**Tác giả:** Arc - Builder (FEOFALLS Team)
**Ngày cập nhật:** 2026-04-01
**Cơ sở:** v1.723 | Thêm [OBSERVATION] type, 3 metadata mới (TOOL_CALLS, SESSION_TOKENS, PARITY_STATUS), Structured Output Retry tracking.

**Mục tiêu chính:**
- Loại bỏ hoàn toàn rủi ro LLM sinh NODE_ID sai format
- Đảm bảo uniqueness của NODE_ID ngay cả khi trùng keyword + ngày (short-hash)
- Bảo vệ causal chain khi consolidate (archived_alias.json – redirect không sửa edge gốc)
- Kiểm soát token & context explosion qua Circuit Breaker tiered expansion
- Tự động semantic summary nodes + archive redirect chống node explosion
- Giữ single source of truth là Markdown files
- **[MỚI v1.9]** Tracking tool calls, session tokens, parity status per memory entry
- Complexity Budget: không tăng so với v1.723 (≤9)

---

## 1. Node ID Generation Rule (BẮT BUỘC – Deterministic & Collision-Resistant)

**Nguyên tắc cốt lõi (giữ nguyên từ v1.722):**
LLM **không được phép** tự viết NODE_ID.
LLM chỉ cung cấp 2 trường bắt buộc:

- **TYPE**: một trong các loại được liệt kê ở phần 2
- **KEYWORD**: chuỗi ngắn gọn (tối đa 5 từ, không chứa ký tự đặc biệt ngoài dấu cách)

**write_helper.py sẽ tự động sinh NODE_ID theo format:**

```
[TYPE]-[YYYY-MM-DD]-[keyword-ngắn-gọn][-short-hash]
```

**Short-hash logic (collision avoidance):**
- Input hash = keyword + date + content_snippet[:20]
- short_hash = sha1(input)[:4] (hex)
- Nếu ID cơ bản đã tồn tại → thêm -short_hash
- Nếu vẫn trùng (hiếm) → fallback -nn-short_hash (nn = 01,02,...)

**Ví dụ:**

LLM viết:
```
TYPE: fact
KEYWORD: gemini benchmark Q1 2026
```

write_helper.py sinh:
```
fact-2026-03-15-gemini-benchmark-q1-2026-a4f3
```

**Lý do:**
- Loại bỏ 100% lỗi format
- Đảm bảo uniqueness cao (4 ký tự hex đủ cho hàng nghìn node/ngày)
- Giữ tính readable
- Shadow Evaluator block write nếu thiếu TYPE hoặc KEYWORD

---

## 2. Memory Type Prefix (BẮT BUỘC – danh sách đầy đủ)

Các prefix hợp lệ (phải viết hoa và nằm trong `[]` ngay đầu entry):

```
[FACT]          – Dữ liệu khách quan, có thể verify
[INSIGHT]       – Bài học chưng cất từ nhiều episode
[ERROR]         – Sai lầm + root cause
[DECISION]      – Quyết định đã thực thi + rationale
[PREFERENCE]    – Giá trị ưu tiên của Creator / người dùng
[WORKFLOW]      – Quy trình lặp lại
[HYPOTHESIS]    – Giả thuyết chưa verify
[EXPERIMENT]    – Kết quả thử nghiệm
[LESSON]        – Bài học rút ra sau verify
[HEURISTIC]     – Rule nhanh, distilled vào Layer 2
[SIGNAL]        – Tín hiệu bất thường (trigger reflection)
[CONSTRAINT]    – Ràng buộc governance
[ASSUMPTION]    – Giả định cần theo dõi
[CRON_LOG]      – [v1.723] Kết quả chạy cron job (nightly/weekly/health)
[OBSERVATION]   – [MỚI v1.9] Feedback thực tế từ Creator / môi trường (dùng để tự calibrate)
```

> **[CRON_LOG]** (v1.723): Ghi lại kết quả tự động hóa. STATE mặc định: ACTIVE. VALID_UNTIL: AUTO (90 ngày). Không cần causal relations trừ khi phát hiện lỗi quan trọng.

> **[OBSERVATION]** (v1.9): Ghi nhận hiện tượng thực tế từ Creator hoặc môi trường mà không cần interpret. Ví dụ: "Creator phản hồi output quá dài", "API response time tăng 200%". STATE mặc định: ACTIVE. VALID_UNTIL: AUTO (90 ngày). Thường trở thành source của CAUSED_BY cho [INSIGHT] hoặc [LESSON] sau Reflection.

---

## 3. Metadata Block (BẮT BUỘC – đặt ngay sau prefix)

Mọi entry phải có đầy đủ các trường sau (dùng dấu `-` đầu dòng):

```markdown
- NODE_ID: fact-2026-04-01-gemini-benchmark-q1-2026-a4f3     # do write_helper.py sinh
- STATE: PINNED | IMMUTABLE | ACTIVE | DECAYING | ARCHIVED | CONSOLIDATED
- VERSION: v1 | v2 | v3 ...
- SUPERSEDES: [NODE_ID cũ]                                    # node bị thay thế (để trống nếu không có)
- SUPERSEDED_BY: [NODE_ID mới]                                # node thay thế hiện tại (để trống nếu không có)
- TIMESTAMP: 2026-04-01                                       # ngày tạo
- LAST_ACCESS: 2026-04-01                                     # nightly tự update khi retrieve
- VALID_UNTIL: 2026-07-01 | INDEFINITE | AUTO                 # AUTO = TIMESTAMP + 90 ngày
- CONFIDENCE: 0.00–1.00                                       # tự đánh giá khi tạo
- SALIENCE: auto                                              # nightly tính: log(access_count + 1)
- SIGNAL: <optional>                                          # ví dụ: repeated_timeout_error
- TAGS: [tag1, tag2, tag3]                                    # optional
- TOOL_CALLS: []                                              # [MỚI v1.9] Danh sách tool calls trong session tạo entry này
- SESSION_TOKENS: 0                                           # [MỚI v1.9] Số tokens đã dùng trong session tạo entry này
- PARITY_STATUS: "OK" | "GAP_LOW" | "GAP_MEDIUM" | "GAP_CRITICAL"  # [MỚI v1.9] Kết quả parity check gần nhất
- STRUCTURED_RETRY_COUNT: 0                                   # [MỚI v1.9] Số lần retry output format sai
```

> **Metadata mới v1.9:** Các trường TOOL_CALLS, SESSION_TOKENS, PARITY_STATUS, STRUCTURED_RETRY_COUNT là **optional**. Nếu agent chưa implement Tool Harness / Session Intelligence thì để giá trị mặc định ([] / 0 / "OK" / 0). Các trường này giúp audit và debug dễ dàng hơn.

**STATE reference:**

| STATE | Ý nghĩa |
|---|---|
| `PINNED` | Quan trọng, ưu tiên cao — không decay |
| `IMMUTABLE` | Bất biến tuyệt đối (chỉ dùng cho Constitution layer) |
| `ACTIVE` | Đang sử dụng bình thường |
| `DECAYING` | Không access 60+ ngày — sắp archive |
| `ARCHIVED` | Đã archive vào `/memory/archive/YYYY-MM/` |
| `CONSOLIDATED` | Đã được tổng hợp vào [INSIGHT]-summary node |

---

## 4. Causal Relations (CHỈ dùng NODE_ID đã được sinh – deterministic)

```markdown
- CAUSED_BY: hypothesis-2026-03-10-qwen-test-a1b2
- RESOLVES: error-2026-03-05-gemini-400-c3d4
- DEPENDS_ON: experiment-2026-03-10-50-tests-e5f6
- PREVENTS: constraint-2026-01-01-no-shared-keys-7890
- TRIGGERED_BY: signal-2026-03-11-repeated-timeout-1234
- IMPACTS: decision-2026-03-11-separate-api-key-abcd
- DERIVED_FROM: insight-2026-03-09-auth-conflict-pattern-ef01
```

**Quy tắc direction safety (Shadow Evaluator enforce):**
- PINNED/IMMUTABLE nodes chỉ được là **target** (không phải source) của các edge mutation
- ARCHIVED nodes: chỉ được trỏ đến qua archived_alias.json (không trỏ trực tiếp)

---

## 5. Confidence Propagation Formula (hardcoded trong query.py)

```latex
DerivedConfidence_B = Confidence_A × EdgeWeight × RetrievalScore × e^(-age / τ)
```

- τ mặc định = 90 ngày (insight công nghệ), 365 ngày (fact dài hạn)
- RetrievalScore: rerank_score normalize [0,1]
- Shadow Evaluator cutoff: Derived Confidence < 0.65 → block cascade + warning

**Edge Weight Table (hardcoded):**

| Relation | Weight |
|---|---|
| CAUSED_BY / RESOLVES | 0.92 |
| DEPENDS_ON / DERIVED_FROM | 0.85 |
| TRIGGERED_BY / PREVENTS | 0.78 |
| IMPACTS | 0.90 |

---

## 6. Circuit Breaker – Tiered Expansion (hardcoded trong query.py)

```text
if initial_rerank_score < 0.5:
    → Hard Stop: "Không tìm thấy thông tin đủ tin cậy (rerank < 0.5)"
elif derived_confidence > 0.75:
    → Selective Expand: max_nodes = 5, depth = 2
elif user_prompt contains "--deep-reasoning":
    → Deep Dive: max_nodes = 8, depth = 2
else:
    → No Expand: chỉ top-k chunks (tiết kiệm token)
```

---

## 7. Lifecycle, Consolidation & Archive Redirect Rules (nightly_indexer.py enforce)

- **PINNED / IMMUTABLE:** miễn nhiễm hoàn toàn (bảo vệ Layer 0–2)
- **ACTIVE → DECAYING:** 60 ngày không access
- **DECAYING → ARCHIVED:** +30 ngày hoặc VALID_UNTIL hết hạn
- **ARCHIVED:** loại khỏi ChromaDB, chuyển file vào `/memory/archive/YYYY-MM/`

**Automatic Semantic Summary Nodes (chống node explosion):**
- Weekly check: nếu cluster DECAYING > 20 node
- Tạo node mới: `[INSIGHT]-YYYY-MM-DD-cluster-<tên_ngắn>`
- Trỏ CAUSED_BY về các node cũ (giữ lịch sử)
- Ghi redirect vào **archived_alias.json**:
  ```json
  {
    "error-2026-03-05-gemini-400": "insight-2026-03-19-auth-cluster-summary",
    "signal-2026-03-11-repeated-timeout": "insight-2026-03-19-auth-cluster-summary"
  }
  ```
- Archive node cũ → không sửa edge gốc
- Reindex chỉ summary node mới

**Query behavior (query.py):**
- Trước khi fetch NODE_ID → check archived_alias.json
- Nếu có alias → redirect + ghi note "[Redirect] node cũ đã consolidate → sử dụng summary"
- Giữ nguyên traceability lịch sử

---

## 8. Retrieval Constraints & Safety Limits (hard limit)

- Max depth = 2
- Max expanded nodes mặc định = 5
- Shadow cutoff: Derived Confidence < 0.65 → block cascade + warning
- Context expansion vượt giới hạn → cắt + warning "Context expansion reached safety limit"

---

## 9. Ví dụ entry đầy đủ (copy-paste template)

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

**Nội dung:**
Gemini v1.5 Pro đạt tỷ lệ pass@1 92% trên codebase nội bộ, vượt Qwen 2.5 Coder (85%).
```

```markdown
[OBSERVATION] Creator phản hồi output quá dài
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

**Nội dung:**
Creator phản hồi rằng output của agent quá dài, cần rút gọn. Đây là observation thực tế, chưa có insight.
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

**Kết quả:**
- Active: 124, Decaying: 3, Archived: 0, Errors: 0
- Consolidation: không cần (decaying < 20)
- Runtime: 03:02 UTC (đúng lịch)
```

---

## 10. Lưu ý quan trọng nhất (v1.9)

- Toàn bộ NODE_ID phải do `write_helper.py` sinh → không chấp nhận NODE_ID tự viết từ LLM
- Source-of-truth duy nhất là Markdown files
- `archived_alias.json` và `causal_index.json` chỉ là derived cache → regenerate được bất kỳ lúc nào
- Shadow Evaluator phải block write nếu thiếu TYPE hoặc KEYWORD
- Graph consistency được bảo vệ bởi `weekly_graph_validator.py`
- **[v1.723]** Thêm type `[CRON_LOG]` để tracking kết quả automation
- **[MỚI v1.9]** Thêm type `[OBSERVATION]` để ghi nhận feedback Creator / môi trường
- **[MỚI v1.9]** 4 metadata mới: TOOL_CALLS, SESSION_TOKENS, PARITY_STATUS, STRUCTURED_RETRY_COUNT
- **[MỚI v1.9]** Các trường metadata mới là **optional** — agents chưa implement Tool Harness/Session Intelligence có thể dùng giá trị mặc định

---

*Arc - Builder | FEOFALLS Team | v1.9 | 2026-04-01*
*Áp dụng cho tất cả Solo Agents trong hệ sinh thái FEOFALLS*

