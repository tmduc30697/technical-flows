# Enhance sequence — Phát hiện dấu hiệu chia sẻ tài khoản qua đa dạng địa lý

Đây là **enhance**, flow hoàn toàn mới chạy khi có login mới, đánh giá mức độ nghi ngờ chia sẻ tài khoản dựa trên vị trí địa lý các session gần đây. Đáp ứng yêu cầu 4: phân biệt hộ gia đình hợp lệ dùng nhiều thiết bị với dấu hiệu chia sẻ vượt phạm vi (nhiều thành phố/quốc gia khác nhau trong cùng khung giờ), áp policy slot chặt hơn cho trường hợp nghi ngờ mà không cản trở người dùng bình thường.

```mermaid
sequenceDiagram
    actor Device as Thiết bị đăng nhập mới
    participant App as Streaming Service
    participant Risk as Risk Assessment Job
    participant DB as Database

    Device->>App: Login thành công (kèm IP, suy ra city/country)
    App->>DB: INSERT/UPDATE DEVICE_SESSION (login_ip, login_city, login_country)

    App->>Risk: Trigger đánh giá risk cho user_id
    Risk->>DB: SELECT DISTINCT login_city, login_country FROM DEVICE_SESSION WHERE user_id=... AND started_at > now() - 24h
    DB-->>Risk: Danh sách city/country distinct trong 24h gần nhất

    alt Các thiết bị cùng thành phố hoặc lân cận (hộ gia đình hợp lệ)
        Risk->>DB: INSERT SHARING_RISK_ASSESSMENT (risk_level=normal, effective_max_devices=max_concurrent_devices)
        Note over Risk,DB: Không thay đổi policy, người dùng hợp lệ không bị ảnh hưởng
    else Nhiều thành phố/quốc gia khác nhau trong cùng khung giờ (nghi ngờ chia sẻ)
        Risk->>DB: INSERT SHARING_RISK_ASSESSMENT (risk_level=suspicious, effective_max_devices=giảm xuống, vd còn 1)
        Note over Risk,DB: Policy slot chặt hơn được áp dụng ngay cho các lần acquire-slot tiếp theo
    end

    Note over App,DB: Lần acquire-slot kế tiếp của user này sẽ so sánh với effective_max_devices thay vì max_concurrent_devices mặc định
```
