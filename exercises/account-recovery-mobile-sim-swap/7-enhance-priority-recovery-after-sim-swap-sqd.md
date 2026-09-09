# Enhance sequence — Priority recovery after SIM-swap

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đáp ứng yêu cầu thứ 5 của đề bài: khi chủ tài khoản thật báo cáo đã bị chiếm quyền qua SIM-swap, cần 1 luồng khôi phục ưu tiên **không phụ thuộc vào số điện thoại đó nữa**, đồng thời khóa/hoàn tác ngay các thay đổi kẻ tấn công đã thực hiện.

```mermaid
sequenceDiagram
    actor User as Chủ tài khoản thật
    participant App as Mobile App / Web support
    participant DB as PRIORITY_RECOVERY_CASE store
    participant Verify as Backup email / Manual ID verification
    actor Support

    User->>App: Báo cáo bị chiếm tài khoản do SIM-swap (qua kênh khác, không dùng số điện thoại cũ)
    App->>DB: Tạo PRIORITY_RECOVERY_CASE (reason=reported_sim_swap, status=pending_verification)
    App->>Verify: Bắt đầu xác minh không phụ thuộc phone — gửi mã tới backup_email đã verify từ trước
    alt Backup email còn quyền truy cập
        User->>Verify: Nhập mã xác minh từ backup email
        Verify-->>DB: verification_method=backup_email, verified
    else Không còn backup email hoặc nghi ngờ thêm
        User->>Support: Cung cấp giấy tờ tùy thân cho xác minh thủ công
        Support->>DB: verification_method=manual_id_verification, verified
    end
    DB->>DB: PRIORITY_RECOVERY_CASE status=verified
    DB->>DB: Tìm toàn bộ SENSITIVE_ACTION_REQUEST đã thực thi trong khoảng thời gian nghi bị chiếm quyền
    DB->>DB: Tạo ROLLED_BACK_ACTION cho từng thay đổi (khôi phục lại email/phương thức khôi phục ban đầu)
    App->>App: Thu hồi toàn bộ SESSION/DEVICE mà kẻ tấn công đã dùng
    App-->>User: Trả lại quyền kiểm soát tài khoản, yêu cầu thiết lập lại thiết bị tin cậy mới
```
