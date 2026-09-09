# Enhance sequence — Rollback (kèm kế hoạch xử lý dữ liệu định dạng mới, ghi audit log)

Đây là **enhance** của flow `rollback` đã có ở base. So với base (chỉ rollback mã nguồn, không có kế hoạch dữ liệu, không ghi log), nay rollback bắt buộc thực thi `ROLLBACK_DATA_PLAN` đã định nghĩa trước để xử lý các dòng dữ liệu đã ghi theo định dạng mới, và mọi bước rollback đều ghi vào `AUDIT_LOG` bất biến. Đáp ứng yêu cầu 1 và 4 của đề bài.

```mermaid
sequenceDiagram
    actor Dev as Kỹ sư triển khai
    participant CI as CI/CD
    participant Canary as Deployment canary (đang chạy)
    participant Stable as Deployment cũ
    participant Plan as ROLLBACK_DATA_PLAN store
    participant DB as Database giao dịch
    participant Audit as AUDIT_LOG (append-only)
    participant Router as Router

    Note over Canary,DB: Deployment canary đã ghi một số TRANSACTION theo định dạng dữ liệu mới trước khi lỗi được phát hiện

    Dev->>CI: Phát hiện lỗi nghiêm trọng, ra lệnh rollback
    CI->>Audit: Ghi AUDIT_LOG(action=rolled_back, performed_by=Dev, previous_traffic_percent=hiện tại, new_traffic_percent=0, lý do lỗi)

    CI->>Plan: Lấy ROLLBACK_DATA_PLAN đã định nghĩa sẵn cho deployment này
    Plan-->>CI: migration_action = downgrade_convert (áp dụng cho các TRANSACTION đã ghi định dạng mới)

    CI->>DB: Chạy script convert ngược các dòng TRANSACTION mới về định dạng bản cũ hiểu được
    DB-->>CI: Đã convert xong toàn bộ dòng bị ảnh hưởng, không mất số liệu tài chính

    CI->>Router: Trỏ 100% traffic về Stable
    CI->>Stable: Xác nhận Stable hoạt động bình thường, đọc đúng dữ liệu đã convert

    CI->>Audit: Ghi AUDIT_LOG bổ sung (action=data_migration_completed, số dòng đã convert, thời điểm hoàn tất)

    Note over Plan,Audit: Nếu deployment tương lai đổi định dạng dữ liệu khác, ROLLBACK_DATA_PLAN phải được cập nhật trước khi canary bắt đầu, không được để tới lúc cần rollback mới nghĩ cách xử lý
```
