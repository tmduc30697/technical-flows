# Enhance sequence — Emergency access (nhân viên y tế cấp cứu)

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đáp ứng yêu cầu 1, 2 và 3 của đề bài: luồng khẩn cấp riêng, không chờ xác minh danh tính đầy đủ của bệnh nhân, nhưng phạm vi bị giới hạn chặt (chỉ dị ứng + thuốc đang dùng), tự hết hạn sau thời gian ngắn, log chi tiết và tự động kích hoạt review sau sự việc.

```mermaid
sequenceDiagram
    actor Staff as Medical Staff (cấp cứu)
    participant App as Healthcare Platform
    participant DB as EMERGENCY_ACCESS_GRANT store
    participant Record as MEDICAL_RECORD store
    participant Log as EMERGENCY_ACCESS_LOG
    participant Review as POST_INCIDENT_REVIEW

    Staff->>App: Yêu cầu xem gấp dị ứng/thuốc của bệnh nhân (kèm justification, không có thời gian chờ xác minh đầy đủ)
    App->>DB: Tạo EMERGENCY_ACCESS_GRANT (scope=allergies+current_medications, status=active, expires_at=now+ngắn)
    App->>Record: Lấy đúng 2 field trong scope (không lấy full_history, không lấy BILLING_INFO)
    Record-->>App: Trả về allergies + current_medications
    App->>Log: Ghi log từng field đã xem (field_viewed, viewed_at)
    App-->>Staff: Hiển thị đúng phạm vi cho phép
    Note over DB: Sau expires_at, EMERGENCY_ACCESS_GRANT tự động status=expired, không gia hạn ngầm
    DB->>Review: Tự động tạo POST_INCIDENT_REVIEW ngay khi grant kết thúc (hết hạn hoặc đóng thủ công)
    Review->>Review: Nhân viên khác (không phải người đã dùng quyền khẩn cấp) rà soát justification + log đã xem
    alt Hợp lệ
        Review->>DB: review_outcome=justified
    else Nghi ngờ lạm dụng
        Review->>DB: review_outcome=flagged_for_investigation
    end
```
