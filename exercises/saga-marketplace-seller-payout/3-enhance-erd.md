# ERD — Enhance (sau khi có saga payout)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 5 yêu cầu của đề bài — verify trạng thái thực qua đối tác trước khi compensate, idempotent transfer reference, saga độc lập theo từng seller trong batch, và event log truy vấn theo seller/đơn hàng. So với base, `PARTNER_TRANSFER` nay có `idempotency_key` cố định và `verified_status` riêng biệt với `status` ban đầu, và mỗi `PAYOUT` gắn với một `SAGA_INSTANCE` độc lập trong `PAYOUT_BATCH`.

```mermaid
erDiagram
    SELLER ||--|| SELLER_BALANCE : has
    SELLER ||--o{ ORDER : fulfills
    SELLER ||--o{ PAYOUT : receives
    PAYOUT ||--o{ ORDER : aggregates
    PAYOUT ||--o| PARTNER_TRANSFER : "sent via"

    PAYOUT_BATCH ||--o{ PAYOUT : contains
    PAYOUT ||--|| SAGA_INSTANCE : "orchestrated by"
    SAGA_INSTANCE ||--o{ SAGA_STEP : consists_of
    SAGA_STEP ||--o{ SAGA_EVENT : logs

    SELLER {
        string seller_id PK
        string business_name
        string bank_account_info
        string status
    }

    SELLER_BALANCE {
        string balance_id PK
        string seller_id FK
        decimal available_amount
    }

    ORDER {
        string order_id PK
        string seller_id FK
        decimal order_amount
        decimal commission_amount
        string status
        datetime completed_at
    }

    PAYOUT_BATCH {
        string batch_id PK
        date period
        int total_sellers
        datetime started_at
    }

    PAYOUT {
        string payout_id PK
        string seller_id FK
        string batch_id FK
        decimal total_amount
        string status
    }

    SAGA_INSTANCE {
        string saga_id PK
        string payout_id FK
        string current_step
        string status
        datetime started_at
    }

    SAGA_STEP {
        string step_id PK
        string saga_id FK
        string name
        string status
        datetime executed_at
    }

    SAGA_EVENT {
        string event_id PK
        string step_id FK
        string seller_id FK
        string order_id FK
        string event_type
        string payload
        datetime created_at
    }

    PARTNER_TRANSFER {
        string transfer_id PK
        string payout_id FK
        string idempotency_key
        decimal amount
        string status
        string verified_status
        datetime initiated_at
        datetime verified_at
    }
```
