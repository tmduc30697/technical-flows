# Enhance ERD — Thêm heartbeat, fencing token, risk detection và kick notification

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, `DEVICE_SESSION` được bổ sung các trường theo dõi sức khỏe kết nối và định danh địa lý, đồng thời có 2 entity mới, ứng trực tiếp với các yêu cầu:

- `DEVICE_SESSION.fencing_token` — hỗ trợ yêu cầu 1 (distributed lock per-user khi giành slot cuối, dùng token tăng dần để phát hiện ghi trễ sau khi lock đã hết hạn). Bản thân distributed lock là coordination tạm thời trên Redis (key `slot-lock:{user_id}`), không phải entity dữ liệu bền vững nên không xuất hiện trong ERD.
- `DEVICE_SESSION.status` mở rộng thêm giá trị `kicked`, cùng entity mới `DEVICE_KICK_EVENT` — đáp ứng yêu cầu 2 (dừng phát ngay và hiển thị lý do cụ thể trên thiết bị bị đá), đồng thời entity này cũng được tái dùng cho yêu cầu 5 (force-logout thủ công).
- `DEVICE_SESSION.last_heartbeat_at` cùng `SUBSCRIPTION_PLAN.heartbeat_interval_sec`/`heartbeat_timeout_sec` — đáp ứng yêu cầu 3 (xác định thiết bị ngừng hoạt động qua timeout thay vì tín hiệu logout rõ ràng).
- `DEVICE_SESSION.login_ip`/`login_city`/`login_country`, cùng entity mới `SHARING_RISK_ASSESSMENT` — đáp ứng yêu cầu 4 (phát hiện dấu hiệu chia sẻ tài khoản vượt phạm vi qua đa dạng địa lý, áp policy slot chặt hơn khi nghi ngờ).

```mermaid
erDiagram
    USER ||--|| SUBSCRIPTION_PLAN : subscribes
    USER ||--o{ DEVICE_SESSION : occupies
    USER ||--o{ SHARING_RISK_ASSESSMENT : "được đánh giá"
    DEVICE_SESSION ||--o{ DEVICE_KICK_EVENT : "có thể bị"

    USER {
        string id PK
        string email
        string subscription_plan_id FK
    }
    SUBSCRIPTION_PLAN {
        string id PK
        string name
        int max_concurrent_devices
        int heartbeat_interval_sec "vd 30"
        int heartbeat_timeout_sec "vd 90"
    }
    DEVICE_SESSION {
        string id PK
        string user_id FK
        string device_id
        string device_name
        string status "active|kicked|ended|expired"
        string fencing_token "tăng dần theo lần acquire lock"
        string login_ip
        string login_city
        string login_country
        datetime started_at
        datetime last_heartbeat_at
    }
    DEVICE_KICK_EVENT {
        string id PK
        string device_session_id FK
        string kicked_by "system|user|new_device"
        string reason "slot_taken_by_new_device|manual_force_logout|risk_policy"
        datetime occurred_at
    }
    SHARING_RISK_ASSESSMENT {
        string id PK
        string user_id FK
        int distinct_cities_count
        int distinct_countries_count
        string risk_level "normal|suspicious"
        int effective_max_devices "policy chặt hơn khi suspicious"
        datetime evaluated_at
    }
```
