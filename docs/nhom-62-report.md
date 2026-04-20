# Day 13 Observability Lab Report

> **Instruction**: Fill in all sections below. This report is designed to be parsed by an automated grading assistant. Ensure all tags (e.g., `[GROUP_NAME]`) are preserved.

## 1. Team Metadata
- [GROUP_NAME]: 62
- [REPO_URL]: https://github.com/dokhiem2k4/Lab13-Observability
- [MEM  - Mem  
  - Member A: [Phan Thanh Sang] | Role: Correlation ID
  - Member B: [Trần Tiến Dũng] | Role: Log Enrichment
  - Member C: [Trần Đình Minh Vương] | Role: Logging & PII
  - Member D: [Đỗ Minh Khiêm] | Role: Tracing + Incident + Metrics
  - Member E: [Ngô Hải Văn] | Role: Dashboard + SLO/Alerts + Demo & Report

---

## 2. Group Performance (Auto-Verified)
- [VALIDATE_LOGS_FINAL_SCORE]: /100
- [TOTAL_TRACES_COUNT]: 
- [PII_LEAKS_FOUND]: 

---

## 3. Technical Evidence (Group)

### 3.1 Logging & Tracing
- [EVIDENCE_CORRELATION_ID_SCREENSHOT]: [Correlation ID in Logs](./EVIDENCE_CORRELATION_ID_SCREENSHOT.png)
- [EVIDENCE_PII_REDACTION_SCREENSHOT]: [PII Redaction in Logs](./EVIDENCE_PII_REDACTION_SCREENSHOT.png)
- [EVIDENCE_TRACE_WATERFALL_SCREENSHOT]: [Path to image]
- [TRACE_WATERFALL_EXPLANATION]: (Briefly explain one interesting span in your trace)

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
- [SCENARIO_NAME]: (e.g., rag_slow)
- [SYMPTOMS_OBSERVED]: 
- [ROOT_CAUSE_PROVED_BY]: (List specific Trace ID or Log Line)
- [FIX_ACTION]: 
- [PREVENTIVE_MEASURE]: 

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
