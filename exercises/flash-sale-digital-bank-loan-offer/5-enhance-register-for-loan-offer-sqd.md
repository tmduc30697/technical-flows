# Sequence - Enhance: Đăng ký vay ưu đãi (giữ vị trí atomic, xác nhận suất bất đồng bộ)

Đây là flow **enhance** của `register-for-loan-offer` (so với base). Khác biệt so với base: đăng ký tách thành 2 bước rõ ràng — (1) giữ vị trí hàng đợi bằng atomic increment ngay khi bấm, không liên quan gì tới việc trừ suất, (2) chỉ sau khi qua kiểm tra tín dụng mới atomic gán 1 trong 1000 `SLOT_ALLOCATION` cho khách. Vì bước 1 dùng atomic increment ở DB/cache, 2 khách bấm cùng lúc luôn nhận được 2 `queue_position` khác nhau, không còn lost update. Đáp ứng **yêu cầu 1 và 3** của đề bài.

```mermaid
sequenceDiagram
    participant C1 as Khách hàng A
    participant C2 as Khách hàng B
    participant BE as Backend API
    participant Counter as QUEUE_COUNTER (atomic)
    participant DB as Database
    participant Credit as Credit Bureau (bên ngoài)

    par 2 khách bấm đăng ký gần như cùng lúc
        C1->>BE: Bấm đăng ký
        BE->>Counter: Atomic increment (INCR / SELECT FOR UPDATE)
        Counter-->>BE: queue_position=999
        BE->>DB: Tạo REGISTRATION(status=queued, queue_position=999)
        BE-->>C1: Đã giữ vị trí #999, đang chờ xác nhận
    and
        C2->>BE: Bấm đăng ký
        BE->>Counter: Atomic increment (INCR / SELECT FOR UPDATE)
        Counter-->>BE: queue_position=1000
        BE->>DB: Tạo REGISTRATION(status=queued, queue_position=1000)
        BE-->>C2: Đã giữ vị trí #1000, đang chờ xác nhận
    end
    Note over Counter: Atomic increment đảm bảo không bao giờ có 2 khách cùng nhận 1 queue_position, kể cả tải cao

    BE->>Credit: Kiểm tra tín dụng khách A (bất đồng bộ, không block request đăng ký)
    Credit-->>BE: Đạt điều kiện
    BE->>DB: Transaction atomic, gán SLOT_ALLOCATION cho registration A, status=slot_granted
    BE-->>C1: Xác nhận đã được cấp suất vay ưu đãi
```
