# Base ERD — Sàn xuyên biên giới trước khi khóa tỷ giá và đối soát chuẩn hóa

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài. Hệ thống đã là 1 sàn thương mại xuyên biên giới có buyer trả bằng tiền tệ nội địa, seller nhận theo tiền tệ của mình, thanh toán qua đối tác quốc tế có callback — nên base cần có Buyer, Seller, Order (mang cả 2 loại tiền tệ), giao dịch thanh toán, và 1 bảng tỷ giá. Điểm mấu chốt của base: `EXCHANGE_RATE` chỉ là **tỷ giá hiện tại (live)**, bị ghi đè liên tục, không có cơ chế snapshot/khóa tỷ giá theo từng đơn hàng — đây chính là lỗ hổng mà enhance phải vá. Base cũng **chưa có** entity nào phục vụ refund theo tỷ giá đã khóa hay đối soát tách riêng theo từng loại tiền tệ.

```mermaid
erDiagram
    BUYER ||--o{ ORDER : "đặt"
    SELLER ||--o{ ORDER : "nhận đơn"
    ORDER ||--|| PAYMENT_TRANSACTION : "thanh toán qua"

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
        decimal seller_amount "tính bằng tỷ giá tại thời điểm tính, không lưu snapshot"
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
        string currency_pair "vd USD_VND"
        decimal rate
        datetime updated_at "bị ghi đè liên tục, không giữ lịch sử theo order"
    }
```
