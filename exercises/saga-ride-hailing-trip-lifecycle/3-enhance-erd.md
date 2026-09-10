# ERD — Enhance (sau khi có saga điều phối vòng đời chuyến đi)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 5 yêu cầu của đề bài — compensate khi tài xế tự hủy, cửa sổ chờ khi mất kết nối, luật ưu tiên cho race condition, trạng thái riêng cho thanh toán thất bại sau khi chuyến đã hoàn thành, và saga state persist để resume khi crash. So với base, `TRIP` nay được dẫn dắt bởi `SAGA_INSTANCE`, và có thêm entity theo dõi tài xế bị loại trừ, sự kiện mất kết nối, và thu nhập tạm cho tài xế khi thanh toán chưa xong.

```mermaid
erDiagram
    RIDER ||--o{ TRIP : requests
    DRIVER ||--o{ TRIP : accepts
    TRIP ||--o| PAYMENT_CHARGE : "settled via"

    TRIP ||--|| SAGA_INSTANCE : "orchestrated by"
    SAGA_INSTANCE ||--o{ SAGA_STEP : consists_of
    SAGA_STEP ||--o{ SAGA_EVENT : logs

    TRIP ||--o{ DRIVER_EXCLUSION : "excludes for rematch"
    TRIP ||--o{ DISCONNECT_EVENT : "may have"
    TRIP ||--o| PROVISIONAL_EARNING : "may credit"
    PAYMENT_CHARGE ||--o{ PAYMENT_RETRY_ATTEMPT : "may retry via"

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

    SAGA_INSTANCE {
        string saga_id PK
        string trip_id FK
        string current_step
        string status
        datetime updated_at
    }

    SAGA_STEP {
        string step_id PK
        string saga_id FK
        string name
        string status
        datetime executed_at
    }

    SAGA_EVENT {
        string event_id PK
        string step_id FK
        string event_type
        string payload
        datetime created_at
    }

    DRIVER_EXCLUSION {
        string exclusion_id PK
        string trip_id FK
        string driver_id FK
        string reason
        datetime created_at
    }

    DISCONNECT_EVENT {
        string event_id PK
        string trip_id FK
        string party
        datetime disconnected_at
        datetime reconnected_at
        string resolution
    }

    PAYMENT_CHARGE {
        string charge_id PK
        string trip_id FK
        decimal amount
        string status
    }

    PAYMENT_RETRY_ATTEMPT {
        string attempt_id PK
        string charge_id FK
        int attempt_number
        string result
        datetime attempted_at
    }

    PROVISIONAL_EARNING {
        string earning_id PK
        string trip_id FK
        string driver_id FK
        decimal amount
        string status
    }
```
