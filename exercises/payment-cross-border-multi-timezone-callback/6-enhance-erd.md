# Enhance ERD — Khóa tỷ giá theo order, refund đúng rate, đối soát theo currency chuẩn UTC

Đây là ERD **sau khi** enhance được áp dụng lên base. `BUYER`, `SELLER`, `ORDER`, `PAYMENT_TRANSACTION`, `EXCHANGE_RATE` giữ nguyên. So với base, có 3 nhóm entity mới, ứng trực tiếp với các yêu cầu trong đề bài:

- `EXCHANGE_RATE_SNAPSHOT` (mới) — khóa và lưu đúng tỷ giá tại thời điểm buyer bấm thanh toán, gắn liền với order đó. Callback dù đến trễ bao lâu cũng đọc lại từ đây thay vì tra tỷ giá hiện tại.
- `REFUND` (mới) — ghi nhận hoàn tiền khi callback thật báo thất bại sau khi order đã tạm chuyển "đã thanh toán" qua polling, số tiền hoàn luôn tính theo `EXCHANGE_RATE_SNAPSHOT` đã khóa, không phải tỷ giá hiện tại lúc hoàn tiền.
- `RECONCILIATION_REPORT` + `RECONCILIATION_DISCREPANCY` (mới) — đối soát tách riêng theo từng `currency_pair`, mốc ngày chuẩn hóa về UTC (`report_date_utc`), phát hiện và gắn cờ các order có sai lệch tỷ giá bất thường để điều tra riêng.

```mermaid
erDiagram
    BUYER ||--o{ ORDER : "đặt"
    SELLER ||--o{ ORDER : "nhận đơn"
    ORDER ||--|| PAYMENT_TRANSACTION : "thanh toán qua"
    ORDER ||--|| EXCHANGE_RATE_SNAPSHOT : "khóa tỷ giá lúc thanh toán"
    ORDER ||--o| REFUND : "có thể được hoàn tiền"
    ORDER ||--o{ RECONCILIATION_DISCREPANCY : "có thể bị gắn cờ sai lệch"
    RECONCILIATION_REPORT ||--o{ RECONCILIATION_DISCREPANCY : "phát hiện trong quá trình đối soát"

    BUYER {
        string id PK
        string name
        string country
        string default_currency
    }
    SELLER {
        string id PK
        string name
        string country
        string payout_currency
    }
    ORDER {
        string id PK
        string buyer_id FK
        string seller_id FK
        string buyer_currency
        string seller_currency
        decimal buyer_amount
        decimal seller_amount "tính theo rate đã khóa trong EXCHANGE_RATE_SNAPSHOT"
        string status "pending | paid | failed | refunded"
        datetime created_at
    }
    PAYMENT_TRANSACTION {
        string id PK
        string order_id FK
        string partner_payment_ref
        string status
        datetime created_at
        datetime updated_at
    }
    EXCHANGE_RATE {
        string id PK
        string currency_pair
        decimal rate
        datetime updated_at
    }
    EXCHANGE_RATE_SNAPSHOT {
        string id PK
        string order_id FK
        string currency_pair
        decimal rate "khóa tại thời điểm buyer bấm thanh toán"
        datetime locked_at
    }
    REFUND {
        string id PK
        string order_id FK
        decimal amount_buyer_currency
        decimal amount_seller_currency
        decimal rate_used "luôn bằng rate đã khóa, không phải rate hiện tại"
        string reason
        datetime created_at
    }
    RECONCILIATION_REPORT {
        string id PK
        string currency_pair "đối soát riêng từng loại tiền tệ"
        date report_date_utc "mốc ngày đã chuẩn hóa UTC"
        decimal total_buyer_amount
        decimal total_seller_amount
        string status
    }
    RECONCILIATION_DISCREPANCY {
        string id PK
        string reconciliation_report_id FK
        string order_id FK
        decimal expected_rate
        decimal actual_rate
        decimal deviation_pct
        datetime flagged_at
    }
```
