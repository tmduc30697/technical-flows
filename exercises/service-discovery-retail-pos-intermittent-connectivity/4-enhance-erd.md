# ERD — Enhance (sau khi có health check theo lịch hoạt động)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng yêu cầu đề bài — lịch hoạt động riêng từng chi nhánh, ba trạng thái thiết bị thay vì nhị phân, grace window trước khi báo động, correlation sự cố mạng theo khu vực, và heartbeat thích ứng qua kênh dự phòng. So với base, `DEVICE.status` nay có 3 giá trị và `HEARTBEAT_LOG` biết phân biệt kênh kết nối (primary/backup).

```mermaid
erDiagram
    STORE ||--o{ DEVICE : owns
    STORE ||--|| OPERATING_SCHEDULE : "has"
    DEVICE ||--o{ HEARTBEAT_LOG : sends
    DEVICE ||--o{ INCIDENT_ALERT : "may raise"
    REGIONAL_INCIDENT ||--o{ INCIDENT_ALERT : groups
    REGIONAL_INCIDENT ||--o{ DEVICE : "correlates affected"

    STORE {
        string store_id PK
        string name
        string region
        string address
    }

    OPERATING_SCHEDULE {
        string schedule_id PK
        string store_id FK
        string open_time
        string close_time
        string holiday_calendar
        int grace_window_minutes
    }

    DEVICE {
        string device_id PK
        string store_id FK
        string status
        string current_link_type
        datetime last_heartbeat_at
        datetime registered_at
    }

    HEARTBEAT_LOG {
        string heartbeat_id PK
        string device_id FK
        string link_type
        datetime received_at
    }

    INCIDENT_ALERT {
        string alert_id PK
        string device_id FK
        string regional_incident_id FK
        string type
        datetime triggered_at
        string acknowledged_by
    }

    REGIONAL_INCIDENT {
        string incident_id PK
        string region
        int affected_device_count
        datetime detected_at
        string status
    }
```
