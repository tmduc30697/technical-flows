# ERD — Base (trước khi có saga đặt combo)

Đây là **base**: mô hình dữ liệu suy luận cho nền tảng đặt tour *trước khi* có saga đặt combo. Đề bài giả định nền tảng đã cho phép đặt riêng lẻ từng phần (vé máy bay, khách sạn, xe thuê) qua 3 service/nhà cung cấp khác nhau — nếu không có sẵn các entity đặt chỗ riêng lẻ này thì "saga đảm bảo tất cả hoặc không có gì cho combo" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ đặt chỗ, không suy diễn thêm các module không liên quan (loyalty, khuyến mãi...).

```mermaid
erDiagram
    CUSTOMER ||--o{ FLIGHT_BOOKING : books
    CUSTOMER ||--o{ HOTEL_BOOKING : books
    CUSTOMER ||--o{ CAR_BOOKING : books

    CUSTOMER {
        string customer_id PK
        string full_name
        string email
    }

    FLIGHT_BOOKING {
        string booking_id PK
        string customer_id FK
        string flight_number
        decimal price
        string status
    }

    HOTEL_BOOKING {
        string booking_id PK
        string customer_id FK
        string hotel_id
        date check_in
        date check_out
        decimal price
        string status
    }

    CAR_BOOKING {
        string booking_id PK
        string customer_id FK
        string car_type
        date pickup_date
        date return_date
        decimal price
        string status
    }
```
