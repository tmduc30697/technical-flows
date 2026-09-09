# Base sequence — Xem hồ sơ bệnh nhân (shared schema, chưa cách ly nghiêm ngặt)

Đây là **base**, flow "Xem hồ sơ bệnh nhân" ở trạng thái hiện tại — lọc theo `tenant_id` trong shared schema, giải mã bằng khóa dùng chung toàn platform, ghi audit log nhưng log này không bị giới hạn xem theo tenant. Flow này liên quan mật thiết tới enhance vì toàn bộ 4 yêu cầu đầu của đề bài (mô hình cách ly, khóa mã hóa riêng, xóa dữ liệu, audit log cách ly) đều nhằm siết chặt đúng những điểm yếu ở đây.

```mermaid
sequenceDiagram
    actor Staff as Nhân viên phòng khám (tenant A)
    participant App as Clinic Records Service
    participant DB as Shared DB (tenant_id column)
    participant Key as ENCRYPTION_KEY (dùng chung)
    participant Log as AUDIT_LOG

    Staff->>App: Yêu cầu xem hồ sơ bệnh nhân X
    App->>DB: SELECT * FROM medical_record WHERE tenant_id = A AND patient_id = X
    DB-->>App: Bản ghi đã mã hóa
    App->>Key: Lấy khóa dùng chung để giải mã
    Key-->>App: key_material (giống hệt khóa dùng cho mọi tenant khác)
    App->>App: Giải mã nội dung hồ sơ
    App->>Log: Ghi AUDIT_LOG(tenant_id=A, staff_user_id, medical_record_id, action=view)
    App-->>Staff: Trả về nội dung hồ sơ

    Note over Log: Log này lưu chung, admin hệ thống có thể query xuyên mọi tenant nếu muốn — chưa có cơ chế chặn
```
