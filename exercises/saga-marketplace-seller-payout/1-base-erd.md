# ERD — Base (trước khi có saga payout)

Đây là **base**: mô hình dữ liệu suy luận cho marketplace *trước khi* có saga điều phối payout. Đề bài giả định sàn đã có `ORDER` hoàn tất, đã tính `SELLER_BALANCE` và đã tích hợp một đối tác thanh toán bên ngoài để chuyển tiền — nếu không có sẵn các entity này thì "saga điều phối trừ số dư và gọi đối tác chuyển tiền" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ payout, không suy diễn thêm các module không liên quan (catalog, review...).

```mermaid
erDiagram
    SELLER ||--|| SELLER_BALANCE : has
    SELLER ||--o{ ORDER : fulfills
    SELLER ||--o{ PAYOUT : receives
    PAYOUT ||--o{ ORDER : aggregates
    PAYOUT ||--o| PARTNER_TRANSFER : "sent via"

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

    PAYOUT {
        string payout_id PK
        string seller_id FK
        decimal total_amount
        string status
    }

    PARTNER_TRANSFER {
        string transfer_id PK
        string payout_id FK
        decimal amount
        string status
        datetime initiated_at
    }
```
