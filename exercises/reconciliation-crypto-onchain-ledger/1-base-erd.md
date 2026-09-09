# Base ERD — Ví on-chain và ledger nội bộ, chưa có đối soát

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có quy trình đối soát định kỳ. Đề bài nói sàn giữ tài sản khách hàng trong ví on-chain (nóng/lạnh) và duy trì ledger nội bộ ghi số dư khả dụng — nên base cần đủ: khách hàng, ví, giao dịch on-chain (nạp/rút) và số dư ledger được cập nhật từ các giao dịch đó. Chưa có entity nào phục vụ đối soát (snapshot theo block height, phát hiện sai lệch, cảnh báo khẩn, báo cáo proof-of-reserves) hay xử lý RBF/fork — những thứ đó là phần enhance.

```mermaid
erDiagram
    CUSTOMER ||--o{ LEDGER_BALANCE : has
    WALLET ||--o{ ON_CHAIN_TRANSACTION : records
    CUSTOMER ||--o{ ON_CHAIN_TRANSACTION : "credited/debited by"

    CUSTOMER {
        string id PK
        string name
    }
    WALLET {
        string id PK
        string wallet_type "hot | cold"
        string address
        string asset_type
    }
    ON_CHAIN_TRANSACTION {
        string id PK
        string wallet_id FK
        string customer_id FK
        string tx_hash
        string direction "deposit | withdrawal"
        string asset_type
        decimal amount
        int block_height
        string status "pending | confirmed"
        datetime created_at
    }
    LEDGER_BALANCE {
        string id PK
        string customer_id FK
        string asset_type
        decimal available_balance
        datetime updated_at
    }
```
