# Enhance ERD — sau khi có quy trình khôi phục khi mất cả 2 kênh

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm entity mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `RECOVERY_CASE` (mới) — hồ sơ khôi phục khi không còn kênh nào khớp, theo dõi trạng thái xuyên suốt quy trình.
- `IDENTITY_EVIDENCE` (mới) — nhiều bằng chứng độc lập (giấy tờ tùy thân, face match...) thay vì 1 câu hỏi bảo mật duy nhất, chống social engineering.
- `SUPPORT_REVIEW` (mới) — quyết định của support agent dựa trên bằng chứng, phục vụ four-eyes/kiểm soát con người.
- `AUDIT_LOG` (mới, append-only/immutable) — ghi lại toàn bộ ai duyệt, dựa trên bằng chứng gì, khi nào — phục vụ compliance khi thanh tra.
- `ACCOUNT_RESTRICTION` (mới) — áp dụng cả lúc chờ xác minh (đóng băng rút/chuyển tiền, vẫn xem được) lẫn cooling-off period sau khi khôi phục thành công (giới hạn rút tiền lớn 24-48h).
- `SESSION` thêm `revoked_at` — phục vụ yêu cầu đăng xuất toàn bộ thiết bị/session cũ sau khôi phục.

```mermaid
erDiagram
    USER ||--|| WALLET : owns
    USER ||--o{ SESSION : creates
    USER ||--o{ RECOVERY_CASE : "opens (khi mất cả 2 kênh)"
    RECOVERY_CASE ||--o{ IDENTITY_EVIDENCE : "collects"
    RECOVERY_CASE ||--o{ SUPPORT_REVIEW : "reviewed by"
    RECOVERY_CASE ||--o{ AUDIT_LOG : "generates"
    USER ||--o{ ACCOUNT_RESTRICTION : "restricted by"

    USER {
        string id PK
        string email
        string phone
        string password_hash
        string kyc_status
        string kyc_document_ref
    }
    WALLET {
        string id PK
        string user_id FK
        decimal balance
    }
    SESSION {
        string id PK
        string user_id FK
        string device_info
        datetime created_at
        datetime expires_at
        datetime revoked_at
    }
    RECOVERY_CASE {
        string id PK
        string user_id FK
        string reason "lost_all_channels"
        string status "pending_verification | frozen | approved | rejected"
        datetime created_at
        datetime resolved_at
    }
    IDENTITY_EVIDENCE {
        string id PK
        string recovery_case_id FK
        string type "id_document | selfie_face_match | other"
        string evidence_ref
        string match_status
        datetime submitted_at
    }
    SUPPORT_REVIEW {
        string id PK
        string recovery_case_id FK
        string reviewer_id
        string decision
        string evidence_considered
        datetime decided_at
    }
    AUDIT_LOG {
        string id PK
        string recovery_case_id FK
        string actor
        string action
        string evidence_ref
        datetime created_at
    }
    ACCOUNT_RESTRICTION {
        string id PK
        string user_id FK
        string type "withdrawal_frozen | view_only | cooling_off_withdrawal_limit"
        string reason
        datetime active_from
        datetime active_until
    }
```
