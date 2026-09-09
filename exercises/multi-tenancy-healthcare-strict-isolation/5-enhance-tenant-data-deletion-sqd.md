# Enhance sequence — Xóa dữ liệu triệt để của một tenant (crypto shredding)

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có quy trình xóa dữ liệu tenant theo yêu cầu pháp lý). Đáp ứng yêu cầu 2 và 3 của đề bài: khi phòng khám ngừng hợp tác hoặc yêu cầu quyền xóa dữ liệu, hủy khóa mã hóa riêng của tenant đó (crypto shredding) và xóa/hủy dữ liệu trên mọi hệ thống lưu trữ trong khung thời gian xác định, có xác nhận hoàn tất.

```mermaid
sequenceDiagram
    actor Admin as Compliance/Tenant Admin
    participant App as Tenant Offboarding Service
    participant Req as TENANT_DELETION_REQUEST
    participant Key as TENANT_ENCRYPTION_KEY
    participant Primary as Primary DB (dedicated tenant A)
    participant Cache as Cache layer
    participant Backup as Backup store
    participant LogStore as Log store

    Admin->>App: Yêu cầu xóa toàn bộ dữ liệu tenant A
    App->>Req: Tạo TENANT_DELETION_REQUEST(status=pending, deadline_at)

    App->>Key: Thu hồi/hủy TENANT_ENCRYPTION_KEY của tenant A
    Key-->>App: status=revoked
    Note over Primary,Backup: Sau khi khóa bị hủy, dữ liệu đã mã hóa ở mọi nơi (kể cả backup cũ) trở thành không thể đọc được — crypto shredding

    App->>Req: Cập nhật status=in_progress
    par Xóa song song trên từng hệ thống lưu trữ
        App->>Primary: Xóa toàn bộ bản ghi tenant A, tạo DELETION_TASK(target=primary_db)
        Primary-->>App: DELETION_TASK.status=done
    and
        App->>Cache: Purge toàn bộ cache key liên quan tenant A, tạo DELETION_TASK(target=cache)
        Cache-->>App: DELETION_TASK.status=done
    and
        App->>Backup: Đánh dấu bản backup chứa tenant A không khôi phục được (khóa đã hủy), tạo DELETION_TASK(target=backup)
        Backup-->>App: DELETION_TASK.status=done
    and
        App->>LogStore: Xóa/ẩn danh log identifiable của tenant A, tạo DELETION_TASK(target=log)
        LogStore-->>App: DELETION_TASK.status=done
    end

    alt Toàn bộ DELETION_TASK hoàn tất trước deadline_at
        App->>Req: Cập nhật status=completed, ghi confirmation_receipt
        App-->>Admin: Xác nhận đã xóa hoàn tất toàn bộ dữ liệu tenant A
    else Có DELETION_TASK quá hạn hoặc thất bại
        App-->>Admin: Cảnh báo vi phạm deadline, cần can thiệp thủ công
    end
```
