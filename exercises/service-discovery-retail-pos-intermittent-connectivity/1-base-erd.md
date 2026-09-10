# ERD — Base (trước khi có health check theo lịch hoạt động)

Đây là **base**: mô hình dữ liệu suy luận cho nền tảng quản lý thiết bị POS *trước khi* có yêu cầu health check thông minh theo lịch hoạt động. Đề bài giả định hệ thống đã có sẵn việc đăng ký thiết bị theo từng chi nhánh và cơ chế heartbeat nhị phân online/offline — nếu chưa có DEVICE/STORE/HEARTBEAT thì việc "phân biệt cửa hàng đóng cửa với thiết bị hỏng" sẽ không có nghĩa. ERD base chưa có lịch hoạt động hay phân loại nhiều trạng thái.

```mermaid
erDiagram
    STORE ||--o{ DEVICE : owns
    DEVICE ||--o{ HEARTBEAT_LOG : sends

    STORE {
        string store_id PK
        string name
        string region
        string address
    }

    DEVICE {
        string device_id PK
        string store_id FK
        string status
        datetime last_heartbeat_at
        datetime registered_at
    }

    HEARTBEAT_LOG {
        string heartbeat_id PK
        string device_id FK
        datetime received_at
    }
```
