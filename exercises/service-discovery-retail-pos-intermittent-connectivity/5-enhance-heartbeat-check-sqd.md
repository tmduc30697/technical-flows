# Sequence Diagram — Enhance: Heartbeat Check

Đây là **enhance**, flow heartbeat check đã đổi hoàn toàn logic so với base: thay vì so với một timeout cố định, hệ thống so thời gian im lặng với `OPERATING_SCHEDULE` riêng của từng chi nhánh, áp dụng grace window, và phân biệt rõ ba trạng thái (đang hoạt động, đang nghỉ theo giờ đóng cửa, mất kết nối bất thường) thay vì chỉ online/offline.

```mermaid
sequenceDiagram
    participant Device as Thiết bị POS
    participant Platform as Nền tảng quản lý POS
    actor Ops as Đội vận hành

    Device--xPlatform: Không gửi heartbeat sau giờ đóng cửa dự kiến
    Platform->>Platform: Tra OPERATING_SCHEDULE của store
    alt đang trong giờ đóng cửa/ngày nghỉ
        Platform->>Platform: Mark DEVICE status=resting, không cảnh báo
    else đã quá giờ mở cửa dự kiến nhưng còn trong grace_window_minutes
        Platform->>Platform: Chờ thêm, không cảnh báo ngay
        Note over Platform: Cho phép nhân viên mở cửa trễ vài phút, đường truyền khởi động chậm
    else quá giờ mở cửa và đã hết grace window
        Platform->>Platform: Mark DEVICE status=disconnected_suspect
        Platform->>Ops: Cảnh báo cần xử lý
    end

    Device->>Platform: Heartbeat trở lại
    Platform->>Platform: Mark DEVICE status=active
```
