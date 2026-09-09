# ERD - Base (tracing thông thường, chưa đạt chuẩn audit)

Đây là trạng thái **base**: hệ thống chuyển tiền giữa các ví điện tử đã có luồng xử lý giao dịch qua `fraud-check`, `ledger-write`, `notification`, và đã dùng distributed tracing (`TRACE`/`SPAN`) như mọi hệ thống khác chỉ để phục vụ debug performance - áp dụng retention/sampling mặc định, span có thể bị ghi đè/sửa, chưa phân biệt external call, chưa có cảnh báo dừng bất thường, chưa đo overhead. Đây là tiền đề khiến enhance cần nâng cấp tracing này lên chuẩn audit trail cho compliance.

```mermaid
erDiagram
    TRANSACTION ||--o| TRACE : "có 1 trace theo dõi hành trình"
    TRACE ||--o{ SPAN : "gồm nhiều span theo từng bước"
    SPAN ||--o{ SPAN : "parent_span_id tự tham chiếu"

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
        int retention_days
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
    }
```
