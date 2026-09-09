# Sequence - Base: Xem số suất còn lại (đọc từ cache trễ)

Đây là flow **base**: khách hàng đang chờ xem dashboard hiển thị số suất còn lại, nhưng dashboard đọc từ 1 lớp cache được refresh định kỳ (vd mỗi 30 giây), không đồng bộ ngay với trạng thái thật trong DB. Khách có thể thấy "còn 5 suất" trong khi thực tế đã hết, hoặc ngược lại. Đây là tiền đề cho yêu cầu 5 của đề bài.

```mermaid
sequenceDiagram
    participant User as Khách hàng đang chờ
    participant Dash as Dashboard
    participant Cache as Cache (refresh mỗi 30s)
    participant DB as Database

    Note over DB: remaining_slots vừa giảm về 0 do các khách khác vừa đăng ký xong
    User->>Dash: Mở dashboard xem số suất còn lại
    Dash->>Cache: Lấy remaining_slots từ cache
    Cache-->>Dash: remaining_slots=5 (dữ liệu cũ từ 25 giây trước)
    Dash-->>User: Hiển thị "còn 5 suất"
    Note over User: Khách tưởng còn cơ hội, tiếp tục chờ dù thực tế đã hết suất từ lâu
```
