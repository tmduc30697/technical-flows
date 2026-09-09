# Enhance ERD — sau khi có circuit breaker chặt bảo vệ core banking

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 3 nhóm entity mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `CIRCUIT_BREAKER_STATE` (mới) — ngưỡng mở thấp hơn bình thường, ưu tiên bảo vệ core banking.
- `RETRY_AUDIT_LOG` (mới) — ghi lại mọi lần tra cứu trạng thái/retry/bỏ qua retry cho giao dịch tiền, phục vụ audit.
- `BREAKER_ESCALATION` (mới) — escalate khẩn tới team vận hành khi breaker mở kéo dài.
- `BALANCE_CACHE` giữ nguyên nhưng nay được dùng chủ động làm fallback có gắn nhãn rõ ràng.

```mermaid
erDiagram
    ACCOUNT ||--o{ TRANSACTION : "performs"
    ACCOUNT ||--o| BALANCE_CACHE : "cached as"
    TRANSACTION ||--o{ RETRY_AUDIT_LOG : "logged via"
    CIRCUIT_BREAKER_STATE ||--o{ BREAKER_ESCALATION : "may trigger"

    ACCOUNT {
        string id PK
        decimal balance
    }
    TRANSACTION {
        string id PK
        string account_id FK
        string type "transfer | payment"
        decimal amount
        string status "pending | success | failed"
        string core_reference_id
    }
    BALANCE_CACHE {
        string account_id PK
        decimal cached_balance
        datetime cached_at
    }
    RETRY_AUDIT_LOG {
        string id PK
        string transaction_id FK
        int attempt_number
        string action "checked_status_before_retry | retried | skipped_retry"
        string result
        datetime logged_at
    }
    CIRCUIT_BREAKER_STATE {
        string id PK
        string service_name "core_banking"
        string state "closed | open | half_open"
        decimal error_rate_threshold "thấp hơn ngưỡng thông thường"
        datetime opened_at
    }
    BREAKER_ESCALATION {
        string id PK
        string service_name FK
        datetime opened_at
        int duration_ms
        string escalation_channel "phone | pager"
        datetime escalated_at
    }
```
