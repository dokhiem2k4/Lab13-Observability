# Day 13 Observability Lab Report

> **Instruction**: Fill in all sections below. This report is designed to be parsed by an automated grading assistant. Ensure all tags (e.g., `[GROUP_NAME]`) are preserved.

## 1. Team Metadata
- [GROUP_NAME]: 62
- [REPO_URL]: https://github.com/dokhiem2k4/Lab13-Observability
- [Menber]:
  - Member A: [Phan Thanh Sang] | Role: Correlation ID
  - Member B: [Trần Tiến Dũng] | Role: Log Enrichment
  - Member C: [Trần Đình Minh Vương] | Role: Logging & PII
  - Member D: [Đỗ Minh Khiêm] | Role: Tracing + Incident + Metrics
  - Member E: [Ngô Hải Văn] | Role: Dashboard + SLO/Alerts + Demo & Report

---

## 2. Group Performance (Auto-Verified)
- [VALIDATE_LOGS_FINAL_SCORE]: 100/100 (PASSED: JSON schema, correlation ID, enrichment, PII scrubbing — chạy `python scripts/validate_logs.py` trên 195 log records)
- [TOTAL_TRACES_COUNT]: 92 (unique correlation IDs phát hiện trong `data/logs.jsonl`; mỗi correlation_id tương ứng 1 trace trên Langfuse)
- [PII_LEAKS_FOUND]: 0 (validate_logs.py không phát hiện `@` hoặc test credit card `4111` trong bất kỳ record nào; 21 lần xuất hiện `[REDACTED_...]` chứng tỏ pipeline scrub đã hoạt động)

---

## 3. Technical Evidence (Group)

### 3.1 Logging & Tracing
- [EVIDENCE_CORRELATION_ID_SCREENSHOT]: [Correlation ID in Logs](./EVIDENCE_CORRELATION_ID_SCREENSHOT.png)
- [EVIDENCE_PII_REDACTION_SCREENSHOT]: [PII Redaction in Logs](./EVIDENCE_PII_REDACTION_SCREENSHOT.png)
- [EVIDENCE_TRACE_WATERFALL_SCREENSHOT]: [Trace Waterfall 1](./EVIDENCE_TRACE_WATERFALL_SCREENSHOT_1.png) & [Trace Waterfall 2](./EVIDENCE_TRACE_WATERFALL_SCREENSHOT_2.png)
- [TRACE_WATERFALL_EXPLANATION]: Một request `/chat` sinh ra trace gồm 3 span lồng nhau: `run` (root, bao toàn bộ `LabAgent.run` trong [app/agent.py:29](../app/agent.py#L29)), span con `retrieve` (mock RAG lookup trong [app/mock_rag.py:16](../app/mock_rag.py#L16)), và span con `generate` (FakeLLM trong [app/mock_llm.py:29](../app/mock_llm.py#L29)). Ở trạng thái bình thường, `retrieve` gần như tức thời (<5ms) và `generate` chiếm ~150ms (do `time.sleep(0.15)` mô phỏng độ trễ LLM), nên waterfall hiển thị `generate` gần như toàn bộ chiều dài thanh của root. Span đáng chú ý là `retrieve`: khi bật incident `rag_slow` (`POST /incidents/rag_slow/enable`), nó nhảy từ vài ms lên ~2500ms ([app/mock_rag.py:20-21](../app/mock_rag.py#L20-L21)) và trở thành khối chiếm ưu thế trên waterfall — đây chính là bằng chứng trực quan để kết luận RAG là nguyên nhân latency tăng thay vì LLM. Metadata đính kèm span (`matched_key`, `doc_count`, `rag_slow`) cho phép điều tra root cause ngay trong UI Langfuse mà không cần mở log.

### 3.2 Dashboard & SLOs
- [DASHBOARD_6_PANELS_SCREENSHOT]: [Dashboard 6 Panels](./DASHBOARD_6_PANELS_SCREENSHOT.png)
- [SLO_TABLE]:
| SLI | Target | Window | Current Value |
|---|---:|---|---:|
| Latency P95 | < 3000ms | 28d | 156ms |
| Error Rate | < 2% | 28d | 0% |
| Cost Budget | < $2.5/day | 1d | $0.0574 |

### 3.3 Alerts & Runbook
- [ALERT_RULES_SCREENSHOT]: [Alert Rules](./ALERT_RULES_SCREENSHOT.png)
- [SAMPLE_RUNBOOK_LINK]: docs/alerts.md#2-high-error-rate

---

## 4. Incident Response (Group)
- [SCENARIO_NAME]: rag_slow
- [SYMPTOMS_OBSERVED]: Sau khi `POST /incidents/rag_slow/enable`, latency của `/chat` nhảy từ baseline ~152ms lên ~2650ms (tăng ~17x). `/metrics` ghi nhận `latency_p50=2652`, `latency_p95=2653`, `latency_p99=2653` (trước đó cả 3 chỉ số đều là 152). `error_breakdown` vẫn `{}` — request không fail mà chỉ chậm, `quality_avg` không đổi (0.8). Nếu để trong dashboard 5 phút, alert `high_latency_p95` ở [config/alert_rules.yaml:2](../config/alert_rules.yaml#L2) sẽ bắn (ngưỡng 5000ms cho tải nhẹ này chưa chạm, nhưng với workload thực + RAG chậm hơn thì dễ vượt).
- [ROOT_CAUSE_PROVED_BY]: Correlation ID mẫu thu được khi bật incident: `req-c0c6ba12`, `req-32f0fcc3`, `req-3f6dc6ab`. Log line trong `data/logs.jsonl`: `event=response_sent correlation_id=req-c0c6ba12 latency_ms=2652`. Trên Langfuse trace waterfall, span `retrieve` ([app/mock_rag.py:16](../app/mock_rag.py#L16)) chiếm ~2500ms trong tổng 2652ms — khớp chính xác với `time.sleep(2.5)` được inject ở [app/mock_rag.py:20-21](../app/mock_rag.py#L20-L21) khi cờ `rag_slow=True`. Span `generate` (FakeLLM) vẫn ~150ms như bình thường, loại trừ LLM là thủ phạm.
- [FIX_ACTION]: Gọi `POST /incidents/rag_slow/disable` để tắt cờ ([app/incidents.py:17](../app/incidents.py#L17)). Request kế tiếp trả về `latency_ms` về lại mức baseline ~150ms; `/metrics` `latency_p95` sẽ giảm dần khi các sample cũ rơi khỏi window. Trong hệ thống thật, tương đương với rollback deploy RAG gây chậm, hoặc scale vector-store / tăng timeout-then-retry.
- [PREVENTIVE_MEASURE]:
  1. Alert rule `high_latency_p95` ([config/alert_rules.yaml:2-11](../config/alert_rules.yaml#L2-L11)) bắn khi `latency_p95_ms > 5000 for 30m`, link tới runbook [docs/alerts.md#1-high-latency-p95](./alerts.md).
  2. Dashboard panel Latency P50/P95/P99 trong [docs/dashboard.html](./dashboard.html) có threshold line SLO 3000ms để nhận ra trước khi alert bắn.
  3. Langfuse trace metadata đính kèm `rag_slow` flag + `doc_count` ([app/mock_rag.py:27-31](../app/mock_rag.py#L27-L31)) giúp oncall phân biệt RAG-slow vs LLM-slow ngay trên UI mà không cần grep log.
  4. Incident injection API (`/incidents/*/enable|disable`) được giữ nguyên để chạy game-day/fire-drill định kỳ, đảm bảo alert + runbook không bị rot.

**Evidence ảnh cần chụp:**
- [INCIDENT_METRICS_BEFORE_AFTER]: `./INCIDENT_METRICS_IMPACT.png` — chụp 2 output `/metrics` cạnh nhau (baseline 152ms vs rag_slow 2652ms).
- [INCIDENT_TRACE_RAG_SLOW]: `./INCIDENT_TRACE_RAG_SLOW.png` — trace waterfall trên Langfuse của 1 trong 3 correlation_id ở trên, thấy rõ span `retrieve` ~2500ms.

---

## 5. Individual Contributions & Evidence

### Phan Thanh Sang
- [TASKS_COMPLETED]: Phụ trách phần Logging, đảm bảo correlation_id được gắn vào logs và thực hiện redaction các thông tin nhạy cảm.
- [EVIDENCE_LINK]: [PR #1](https://github.com/dokhiem2k4/Lab13-Observability/pull/1)

### Trần Tiến Dũng
- [TASKS_COMPLETED]: |
  1. Implement log enrichment cho endpoint `POST /chat` bằng `bind_contextvars(...)` trong `app/main.py`, đảm bảo mọi log `service="api"` có đủ ngữ cảnh truy vết.
  2. Enrich đầy đủ các field theo yêu cầu: `user_id_hash` (hash từ `user_id`), `session_id`, `feature`, `model` (từ `agent.model`), `env` (từ `APP_ENV`).
  3. Fix lỗi gọi `agent.run(...)` (loại bỏ tham số `correlation_id` không đúng signature) để endpoint chạy ổn định và sinh log đúng.
  4. Tạo log thực tế và chạy `python scripts/validate_logs.py` — phần log enrichment PASS (`Records with missing enrichment (context): 0`).
- [EVIDENCE_LINK]: |
  - Branch: `log_enrichment`
  - Commit: `83654bb` (Log enrichment for /chat)
  - Log evidence: `data/logs.jsonl` (các event `request_received`/`response_sent` có đủ `user_id_hash`, `session_id`, `feature`, `model`, `env`)
  - Screenshot paths (to be added): `docs/EVIDENCE_LOG_ENRICHMENT_SCREENSHOT.png`, `docs/EVIDENCE_VALIDATE_LOGS_ENRICHMENT.png`

### [MEMBER_C_NAME]
### Trần Đình Minh Vương
- [TASKS_COMPLETED]: |
  1. Bật `scrub_event` trong structlog pipeline (`app/logging_config.py`), đảm bảo mọi log record đi qua PII redaction trước khi ghi ra file.
  2. Bổ sung pattern PII còn thiếu trong `app/pii.py`: `passport` (hộ chiếu dạng B1234567, AB1234567) và `address_vn` (từ khóa địa chỉ Việt Nam: đường, phường, xã, quận, thôn, ấp, hẻm, ngõ…).
  3. Fix thứ tự pattern `PII_PATTERNS` để tránh `phone_vn` nhận nhầm CCCD 12 chữ số (commit `eb88b78`).
  4. Kiểm tra `data/logs.jsonl` — log đã sạch, không còn lộ email, phone, CCCD, credit card; các giá trị nhạy cảm được thay bằng `[REDACTED_...]`.
  5. Chạy `python scripts/validate_logs.py` — phần PII scrubbing PASS (0 leak phát hiện).
- [EVIDENCE_LINK]: [PR #7](https://github.com/dokhiem2k4/Lab13-Observability/pull/7)

### [MEMBER_D_NAME]
### Đỗ Minh Khiêm
- [TASKS_COMPLETED]: |
  1. Tích hợp Langfuse tracing cho endpoint `/chat` ([app/tracing.py](../app/tracing.py), [app/agent.py](../app/agent.py)): tạo span cho request chính, RAG retrieval và LLM generation; xác thực trace hiển thị đầy đủ trên Langfuse UI.
  2. Xây dựng metrics pipeline ([app/metrics.py](../app/metrics.py)): đếm traffic, tính latency P50/P95/P99, tổng hợp cost/tokens, error_breakdown, quality_avg; expose qua `GET /metrics`.
  3. Hiện thực 3 kịch bản incident injection ([app/incidents.py](../app/incidents.py)): `rag_slow`, `tool_fail`, `cost_spike` kèm endpoint `POST /incidents/{name}/enable|disable`.
  4. Chạy thử các incident, thu thập bằng chứng trace waterfall + /metrics trước/sau để chứng minh tác động lên P95 latency, error rate và cost.
  5. Viết phần Incident Response (Section 4) và Trace Waterfall Explanation trong report.
- [EVIDENCE_LINK]: [PR tracing + metrics + incidents](https://github.com/dokhiem2k4/Lab13-Observability/pull/2)

### Ngô Hải Văn
- [TASKS_COMPLETED]: |
  1. Xây dựng dashboard 6 panel (docs/dashboard.html) đọc real-time từ /metrics endpoint, auto-refresh 15s, có SLO threshold line trên từng panel.
  2. Tạo Grafana-compatible JSON (config/grafana_dashboard.json) với đầy đủ 6 panel: Latency P50/P95/P99, Traffic, Error Rate, Cost, Tokens In/Out, Quality Score.
  3. Rà soát và annotate SLO (config/slo.yaml) với observed values và alert cross-reference.
  4. Bổ sung alert rule thứ 4 (quality_score_degradation P3) và thêm rationale + SLO link vào 4 rules (config/alert_rules.yaml).
  5. Hoàn thiện runbook cho cả 4 alerts với diagnostic steps và mitigation (docs/alerts.md).
  6. Fix Python 3.9 compatibility cho app/schemas.py (Optional[str] thay str|None).
  7. Điền report nhóm (docs/nhom-62-report.md) và viết demo script (docs/demo-script.md).
- [EVIDENCE_LINK]: https://github.com/dokhiem2k4/Lab13-Observability/pull/5

NK]: 
Bonus Items (Optional)
- [BONUS_COST_OPTIMIZATION]: (Description + Evidence)
- [BONUS_AUDIT_LOGS]: (Description + Evidence)
- [BONUS_CUSTOM_METRIC]: (Description + Evidence)
