# Base sequence — Ingest metric (chỉ partition theo thời gian)

Đây là **base**, flow ghi 1 điểm dữ liệu metric ở trạng thái hiện tại — router chỉ xác định `time_bucket` hiện tại rồi ghi vào đúng 1 partition dùng chung cho *mọi* service. Flow này là tiền đề trực tiếp cho yêu cầu 1 và 4 của đề bài, vì đây là nơi phát sinh rủi ro "hot partition" khi một service đẩy metrics dồn dập.

```mermaid
sequenceDiagram
    actor SvcA as Service A (bình thường)
    actor SvcB as Service B (đẩy metrics dồn dập)
    participant Router as Ingest Router
    participant Part as PARTITION (time_bucket=09:00-10:00, node duy nhất)

    SvcA->>Router: Ghi metric point (ts=09:15)
    Router->>Router: Xác định time_bucket=09:00-10:00
    Router->>Part: Ghi vào partition theo time_bucket
    Part-->>Router: OK

    loop Hàng nghìn điểm dữ liệu mỗi giây
        SvcB->>Router: Ghi metric point (ts=09:16)
        Router->>Router: Xác định time_bucket=09:00-10:00 (giống Service A)
        Router->>Part: Ghi vào cùng partition với Service A
    end
    Note over Part: Toàn bộ ghi của Service B dồn vào đúng 1 partition đang phục vụ luôn cả Service A, node xử lý partition này trở thành hot partition, độ trễ ghi của Service A cũng bị ảnh hưởng lây
```
