# Sequence - Enhance: Xem số suất còn lại và vị trí hàng đợi real-time

Đây là flow **enhance** của `view-remaining-slots-dashboard` (so với base). Khác biệt so với base: dashboard không còn đọc từ cache refresh định kỳ, mà nhận cập nhật qua event (đăng ký mới, nhả suất, cấp suất) được phát ngay khi trạng thái trong DB thay đổi, đồng thời khách hàng đang chờ có thể xem đúng vị trí hàng đợi của mình so với suất đã cấp xong. Đáp ứng **yêu cầu 5** của đề bài.

```mermaid
sequenceDiagram
    participant User as Khách hàng đang chờ (#1001)
    participant Dash as Dashboard
    participant BE as Backend API
    participant DB as Database
    participant Bus as Event Bus

    User->>Dash: Mở dashboard, subscribe cập nhật real-time
    Dash->>BE: Lấy trạng thái hiện tại (remaining_slots, queue_position của #1001)
    BE->>DB: Đếm SLOT_ALLOCATION đang active, đếm registration status=queued phía trước #1001
    DB-->>BE: remaining_slots=0, còn 3 người phía trước đang chờ kết quả tín dụng
    BE-->>Dash: remaining_slots=0, vị trí trước #1001 là 3
    Dash-->>User: Hiển thị "Đã hết suất mới, còn 3 người đang xử lý phía trước bạn"

    Note over DB: Khách #999 bị từ chối tín dụng, suất được nhả và cấp cho #1001
    DB->>Bus: Phát event slot_released_and_reassigned(registration=#1001)
    Bus->>Dash: Đẩy cập nhật ngay lập tức
    Dash-->>User: Cập nhật real-time "Chúc mừng, bạn đã được cấp suất vay ưu đãi"
    Note over User: Không còn tình trạng thấy số liệu cache trễ gây hiểu nhầm đã hết suất trong khi thực ra vừa được cấp
```
