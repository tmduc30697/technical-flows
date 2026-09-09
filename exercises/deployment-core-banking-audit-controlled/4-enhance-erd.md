# Enhance ERD — Canary có kiểm soát rủi ro giao dịch, phê duyệt four-eyes, audit log bất biến, đối soát song song

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 5 nhóm thay đổi chính, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `CANARY_STAGE` (mới) — mỗi deployment canary trải qua nhiều giai đoạn, mỗi giai đoạn giới hạn `allowed_transaction_types` (bắt đầu chỉ `balance_inquiry`, sau mới mở rộng sang `transfer`) thay vì chia đều theo tỷ lệ traffic ngẫu nhiên — đáp ứng yêu cầu 2.
- `APPROVAL` (mới) — mỗi `CANARY_STAGE` cần tối thiểu 2 approval từ 2 người khác nhau trước khi chuyển sang giai đoạn kế tiếp hoặc mở lên 100% — đáp ứng yêu cầu 5 (four-eyes).
- `AUDIT_LOG` (mới) — ghi lại mọi hành động trên deployment (bắt đầu canary, tăng tỷ lệ/mở rộng loại giao dịch, phê duyệt, rollback), không thể sửa/xoá (append-only) — đáp ứng yêu cầu 1.
- `SHADOW_COMPARISON` (mới) — mỗi `TRANSACTION` liên quan tới tính toán số dư/giao dịch có thể được gửi song song tới cả bản stable và canary, so sánh kết quả, chỉ trả về kết quả bản đang chính thức — đáp ứng yêu cầu 3.
- `ROLLBACK_DATA_PLAN` (mới) — định nghĩa trước cách xử lý dữ liệu đã ghi theo định dạng mới khi phải rollback (convert ngược, dual-write tạm thời, hay từ chối đọc) — đáp ứng yêu cầu 4.

```mermaid
erDiagram
    BANKING_SERVICE ||--o{ DEPLOYMENT : has
    DEPLOYMENT ||--o{ CANARY_STAGE : "trải qua"
    CANARY_STAGE ||--o{ APPROVAL : "cần tối thiểu 2"
    DEPLOYMENT ||--o{ AUDIT_LOG : "mọi thay đổi được ghi lại"
    DEPLOYMENT ||--o{ ROLLBACK_DATA_PLAN : "định nghĩa trước cách xử lý dữ liệu khi rollback"
    ACCOUNT ||--o{ TRANSACTION : "thực hiện"
    DEPLOYMENT ||--o{ TRANSACTION : "xử lý bởi (stable hoặc canary)"
    TRANSACTION ||--o| SHADOW_COMPARISON : "có thể được đối soát song song"

    BANKING_SERVICE {
        string id PK
        string name "core-banking-transaction-service"
    }
    DEPLOYMENT {
        string id PK
        string service_id FK
        string version
        string role "stable | canary"
        string status "active | rolled_back"
        datetime deployed_at
    }
    CANARY_STAGE {
        string id PK
        string deployment_id FK
        int stage_no
        int traffic_percent
        string allowed_transaction_types "vd: [balance_inquiry] rồi [balance_inquiry,transfer]"
        string status "pending_approval | active | completed | rolled_back"
    }
    APPROVAL {
        string id PK
        string canary_stage_id FK
        string approver_id
        string approver_role
        string decision "approved | rejected"
        datetime approved_at
    }
    AUDIT_LOG {
        string id PK
        string deployment_id FK
        string action "stage_started | traffic_increased | approved | rolled_back"
        string performed_by
        int previous_traffic_percent
        int new_traffic_percent
        string transaction_types_scope
        datetime occurred_at
        boolean immutable "true, append-only"
    }
    ROLLBACK_DATA_PLAN {
        string id PK
        string deployment_id FK
        string new_format_field
        string migration_action "downgrade_convert | dual_write | reject_read"
        datetime executed_at
    }
    ACCOUNT {
        string id PK
        string owner_name
        decimal balance
    }
    TRANSACTION {
        string id PK
        string account_id FK
        string type "balance_inquiry | transfer"
        decimal amount
        string status "pending|success|failed"
        string processed_by_deployment_id FK
        datetime created_at
    }
    SHADOW_COMPARISON {
        string id PK
        string transaction_id FK
        string stable_result
        string canary_result
        boolean is_mismatch
        datetime compared_at
    }
```
