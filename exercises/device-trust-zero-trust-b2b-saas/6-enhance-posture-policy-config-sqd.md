# Enhance sequence — IT admin cấu hình posture policy riêng theo phòng ban

Đây là **enhance**, flow hoàn toàn mới cho phép IT admin định nghĩa và cập nhật `POSTURE_POLICY` theo từng `DEPARTMENT` thay vì dùng một chính sách cứng chung cho toàn công ty. Đáp ứng yêu cầu 4: chính sách posture (phiên bản OS tối thiểu, bắt buộc mã hóa...) phải cấu hình được theo nhóm người dùng/phòng ban khác nhau — ví dụ phòng Tài chính yêu cầu chuẩn khắt khe hơn phòng Marketing.

```mermaid
sequenceDiagram
    actor Admin as IT Admin
    participant App as Internal Tool (Admin Console)
    participant DB as Database

    Admin->>App: Mở màn hình cấu hình posture policy
    App->>DB: SELECT POSTURE_POLICY theo từng DEPARTMENT
    DB-->>App: Danh sách policy hiện tại theo phòng ban
    App-->>Admin: Hiển thị bảng cấu hình theo phòng ban

    Admin->>App: Cập nhật policy cho phòng "Tài chính" (min_os_version cao hơn, require_disk_encryption=true, block_jailbroken_rooted=true)
    App->>DB: UPDATE POSTURE_POLICY WHERE department_id=finance
    DB-->>App: Cập nhật thành công

    Admin->>App: Cập nhật policy cho phòng "Marketing" (min_os_version thấp hơn, require_disk_encryption=true)
    App->>DB: UPDATE POSTURE_POLICY WHERE department_id=marketing
    DB-->>App: Cập nhật thành công

    App-->>Admin: Xác nhận đã lưu, policy có hiệu lực ngay cho các lần login/verify posture kế tiếp của từng phòng ban

    Note over App,DB: Các flow login và posture revocation ở trên đều tra POSTURE_POLICY theo department_id của user, không dùng policy cứng chung
```
