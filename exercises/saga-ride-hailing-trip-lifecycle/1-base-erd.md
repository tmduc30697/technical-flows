# ERD — Base (trước khi có saga điều phối vòng đời chuyến đi)

Đây là **base**: mô hình dữ liệu suy luận cho app gọi xe *trước khi* có saga điều phối. Đề bài giả định hệ thống đã có matching tài xế, trạng thái chuyến đi qua các bước cơ bản, và thanh toán cuối chuyến — nếu không có sẵn `TRIP` với các trạng thái này thì "saga điều phối vòng đời chuyến đi xuyên matching/trip/payment" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ vòng đời chuyến đi, không suy diễn thêm các module không liên quan (khuyến mãi, đánh giá...).

```mermaid
erDiagram
    RIDER ||--o{ TRIP : requests
    DRIVER ||--o{ TRIP : accepts
    TRIP ||--o| PAYMENT_CHARGE : "settled via"

    RIDER {
        string rider_id PK
        string full_name
        string phone
    }

    DRIVER {
        string driver_id PK
        string full_name
        string phone
        string status
    }

    TRIP {
        string trip_id PK
        string rider_id FK
        string driver_id FK
        string status
        datetime requested_at
        datetime matched_at
        datetime picked_up_at
        datetime completed_at
    }

    PAYMENT_CHARGE {
        string charge_id PK
        string trip_id FK
        decimal amount
        string status
    }
```
