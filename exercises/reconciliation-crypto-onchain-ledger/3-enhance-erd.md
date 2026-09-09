# Enhance ERD — sau khi có đối soát định kỳ ví on-chain vs ledger

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 5 nhóm thay đổi, ứng trực tiếp với các yêu cầu trong đề bài:

- `ON_CHAIN_TRANSACTION` bổ sung `confirmations_count`, `is_replaced`, `is_reorged`, `superseded_by_tx_hash` — phân biệt rõ pending/confirmed và phát hiện giao dịch bị RBF/đảo do fork.
- `CONFIRMATION_POLICY` (mới) — ngưỡng xác nhận an toàn riêng theo từng loại tài sản, thay cho ngưỡng cố định chung.
- `RECONCILIATION_RUN` (mới) — 1 lần đối soát tại 1 block height/thời điểm xác định, so khớp tổng on-chain với snapshot ledger cùng thời điểm.
- `RECONCILIATION_DISCREPANCY` (mới) — sai lệch phát hiện được, phân loại mức độ nghiêm trọng.
- `URGENT_ALERT` (mới) — cảnh báo khẩn khi ledger vượt tài sản thực, không chờ lịch định kỳ.
- `PROOF_OF_RESERVES_REPORT` (mới) — báo cáo minh bạch cho khách hàng/cơ quan quản lý.

```mermaid
erDiagram
    CUSTOMER ||--o{ LEDGER_BALANCE : has
    WALLET ||--o{ ON_CHAIN_TRANSACTION : records
    CUSTOMER ||--o{ ON_CHAIN_TRANSACTION : "credited/debited by"
    ON_CHAIN_TRANSACTION }o--|| CONFIRMATION_POLICY : "governed by (per asset_type)"
    RECONCILIATION_RUN ||--o{ RECONCILIATION_DISCREPANCY : detects
    RECONCILIATION_DISCREPANCY ||--o| URGENT_ALERT : "may trigger"
    RECONCILIATION_RUN ||--o| PROOF_OF_RESERVES_REPORT : "can generate"

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
        int confirmations_count
        string status "pending | confirmed | replaced | reorged"
        boolean is_replaced
        boolean is_reorged
        string superseded_by_tx_hash
        datetime created_at
    }
    LEDGER_BALANCE {
        string id PK
        string customer_id FK
        string asset_type
        decimal available_balance
        datetime updated_at
    }
    CONFIRMATION_POLICY {
        string id PK
        string asset_type
        int required_confirmations
    }
    RECONCILIATION_RUN {
        string id PK
        int block_height
        datetime block_time
        decimal onchain_total_balance
        decimal ledger_total_balance
        decimal discrepancy_amount
        string status "matched | discrepancy_detected"
        datetime run_at
    }
    RECONCILIATION_DISCREPANCY {
        string id PK
        string reconciliation_run_id FK
        string asset_type
        string discrepancy_type "ledger_exceeds_onchain | onchain_exceeds_ledger | minor_variance"
        decimal amount
        string severity "critical | warning"
        datetime detected_at
    }
    URGENT_ALERT {
        string id PK
        string discrepancy_id FK
        datetime triggered_at
        string status "open | acknowledged | resolved"
    }
    PROOF_OF_RESERVES_REPORT {
        string id PK
        string reconciliation_run_id FK
        datetime generated_at
        string published_reference
        string requested_by "regulator | customer | internal"
    }
```
