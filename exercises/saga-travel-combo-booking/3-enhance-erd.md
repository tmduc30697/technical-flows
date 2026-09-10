# ERD — Enhance (sau khi có saga đặt combo)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 5 yêu cầu của đề bài — combo gộp 3 phần đặt riêng lẻ, thứ tự đặt theo rủi ro, timeout riêng từng bước, phí hủy khi compensate, trạng thái real-time, và phát hiện saga kẹt. So với base, các booking riêng lẻ không đổi cấu trúc nhưng nay đều thuộc về một `COMBO_BOOKING` và được saga điều phối.

```mermaid
erDiagram
    CUSTOMER ||--o{ COMBO_BOOKING : books
    COMBO_BOOKING ||--o| FLIGHT_BOOKING : includes
    COMBO_BOOKING ||--o| HOTEL_BOOKING : includes
    COMBO_BOOKING ||--o| CAR_BOOKING : includes

    COMBO_BOOKING ||--|| SAGA_INSTANCE : "orchestrated by"
    SAGA_INSTANCE ||--o{ SAGA_STEP : consists_of
    SAGA_STEP |o--o| CANCELLATION_FEE : "may incur"

    CUSTOMER {
        string customer_id PK
        string full_name
        string email
    }

    COMBO_BOOKING {
        string combo_id PK
        string customer_id FK
        decimal total_price
        string status
        datetime created_at
    }

    FLIGHT_BOOKING {
        string booking_id PK
        string combo_id FK
        string flight_number
        decimal price
        string status
    }

    HOTEL_BOOKING {
        string booking_id PK
        string combo_id FK
        string hotel_id
        date check_in
        date check_out
        decimal price
        string status
    }

    CAR_BOOKING {
        string booking_id PK
        string combo_id FK
        string car_type
        date pickup_date
        date return_date
        decimal price
        string status
    }

    SAGA_INSTANCE {
        string saga_id PK
        string combo_id FK
        string current_step
        string status
        datetime last_progress_at
    }

    SAGA_STEP {
        string step_id PK
        string saga_id FK
        string name
        string status
        int timeout_seconds
        datetime executed_at
    }

    CANCELLATION_FEE {
        string fee_id PK
        string step_id FK
        decimal fee_amount
        decimal refunded_amount
        string reason
    }
```
