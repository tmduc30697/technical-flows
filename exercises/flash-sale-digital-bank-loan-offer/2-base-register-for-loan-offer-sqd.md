# Sequence - Base: Đăng ký vay ưu đãi (trừ suất đọc-rồi-ghi, chặn đồng bộ)

Đây là flow **base**: khách hàng bấm đăng ký, backend đọc `remaining_slots`, kiểm tra còn suất thì trừ đi 1 rồi mới gọi kiểm tra tín dụng, toàn bộ request bị block chờ credit bureau trả kết quả (vài giây) trước khi phản hồi khách hàng. Vì đọc-rồi-ghi không atomic, khi nhiều khách bấm cùng lúc có thể xảy ra lost update (2 khách cùng đọc `remaining_slots=1`, cùng trừ xuống 0, cả 2 đều được xác nhận dù chỉ còn đúng 1 suất). Đây là tiền đề cho yêu cầu 1 và 3 của đề bài.

```mermaid
sequenceDiagram
    participant C1 as Khách hàng A
    participant C2 as Khách hàng B
    participant BE as Backend API
    participant DB as Database
    participant Credit as Credit Bureau (bên ngoài)

    par 2 khách bấm đăng ký gần như cùng lúc
        C1->>BE: Bấm đăng ký
        BE->>DB: Đọc remaining_slots (=1)
    and
        C2->>BE: Bấm đăng ký
        BE->>DB: Đọc remaining_slots (=1, chưa thấy thay đổi)
    end
    Note over BE: Cả 2 request đều thấy remaining_slots=1, đều nghĩ mình còn suất
    BE->>DB: Ghi remaining_slots=0 (cho khách A)
    BE->>DB: Ghi remaining_slots=0 (cho khách B, ghi đè, không phát hiện xung đột)
    Note over DB: Lost update, thực tế 2 khách cùng được trừ suất dù chỉ có 1 suất còn lại
    BE->>Credit: Gọi kiểm tra tín dụng khách A (đồng bộ, chờ vài giây)
    Credit-->>BE: Đạt điều kiện
    BE-->>C1: Đăng ký thành công
    BE->>Credit: Gọi kiểm tra tín dụng khách B (đồng bộ, chờ vài giây)
    Credit-->>BE: Đạt điều kiện
    BE-->>C2: Đăng ký thành công
    Note over DB: 2 khách cùng được cấp suất dù chương trình chỉ còn 1, vượt quá total_slots
```
