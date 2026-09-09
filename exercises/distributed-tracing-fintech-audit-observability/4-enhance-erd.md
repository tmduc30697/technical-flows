# ERD - Enhance (tracing nâng cấp thành audit trail cho fintech)

Đây là trạng thái **enhance**: giữ nguyên `TRANSACTION`, `TRACE`, `SPAN` từ base, thêm/sửa:
- `TRACE` thêm `is_compliance_trace` và liên kết `TRACE_RETENTION_POLICY` để có retention dài hơn trace thông thường, không bị xoá theo policy sampling/rotation mặc định - đáp ứng yêu cầu 1. Thêm `business_latency_ms`/`tracing_overhead_ms`/`overhead_ratio_pct` để đo tỉ lệ overhead do tracing gây ra so với latency gốc - đáp ứng yêu cầu 5.
- `SPAN` thêm `is_append_only` và `integrity_hash` để đảm bảo span kiểm tra fraud không sửa được sau khi ghi - đáp ứng yêu cầu 2; thêm `is_external_call`/`external_partner_name` để đánh dấu rõ bước gọi ngân hàng đối tác, tách biệt khỏi SLA nội bộ - đáp ứng yêu cầu 3.
- `TRACE_RETENTION_POLICY`: bảng chính sách retention, phân biệt COMPLIANCE (retention dài, không tự xoá) và STANDARD (retention mặc định) - đáp ứng yêu cầu 1.
- `STALL_ALERT`: cảnh báo tự động khi trace "dừng bất thường" giữa chừng, không có span tiếp theo trong X giây kể từ span cuối - đáp ứng yêu cầu 4.

```mermaid
erDiagram
    TRANSACTION ||--o| TRACE : "có 1 trace theo dõi hành trình"
    TRACE ||--o{ SPAN : "gồm nhiều span theo từng bước"
    SPAN ||--o{ SPAN : "parent_span_id tự tham chiếu"
    TRACE_RETENTION_POLICY ||--o{ TRACE : "áp dụng retention cho"
    TRACE ||--o{ STALL_ALERT : "phát sinh cảnh báo nếu dừng bất thường"

    TRANSACTION {
        string transaction_id PK
        string from_wallet_id
        string to_wallet_id
        decimal amount
        string status
        datetime created_at
    }
    TRACE {
        string trace_id PK
        string transaction_id FK
        string retention_policy_id FK
        boolean is_compliance_trace
        int business_latency_ms
        int tracing_overhead_ms
        decimal overhead_ratio_pct
        datetime started_at
    }
    SPAN {
        string span_id PK
        string trace_id FK
        string parent_span_id FK
        string service_name
        string operation_name
        datetime start_time
        datetime end_time
        string status
        boolean is_append_only
        string integrity_hash
        boolean is_external_call
        string external_partner_name
    }
    TRACE_RETENTION_POLICY {
        string policy_id PK
        string applies_to
        int retention_days
        boolean deletion_exempt
    }
    STALL_ALERT {
        string alert_id PK
        string trace_id FK
        string last_span_id FK
        int seconds_since_last_span
        datetime detected_at
        string status
    }
```
