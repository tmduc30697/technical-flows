# Sequence Diagram — Enhance: Detect Regional Outage

Đây là **enhance**, flow hoàn toàn mới xử lý trường hợp hàng loạt thiết bị cùng khu vực mất kết nối đồng thời — gộp thành một `REGIONAL_INCIDENT` duy nhất thay vì tạo hàng trăm cảnh báo riêng lẻ, đồng thời vẫn phải tách được cảnh báo thật (1 chi nhánh hỏng máy thật giữa lúc có sự cố diện rộng).

```mermaid
sequenceDiagram
    participant Platform as Nền tảng quản lý POS
    actor Ops as Đội vận hành

    Platform->>Platform: Ghi nhận nhiều DEVICE cùng region chuyển disconnected_suspect trong thời gian ngắn
    Platform->>Platform: So sánh số lượng thiết bị mất kết nối với ngưỡng correlation theo region

    alt vượt ngưỡng correlation
        Platform->>Platform: Tạo REGIONAL_INCIDENT (region, affected_device_count)
        Platform->>Platform: Gộp toàn bộ INCIDENT_ALERT liên quan vào regional_incident_id
        Platform->>Ops: Gửi 1 cảnh báo duy nhất, kèm danh sách thiết bị ảnh hưởng
    else dưới ngưỡng, chỉ 1 vài thiết bị lẻ tẻ
        Platform->>Ops: Gửi cảnh báo riêng cho từng thiết bị như bình thường
        Note over Platform,Ops: Đây là trường hợp nghi ngờ hỏng thật, không bị gộp và bỏ sót
    end
```
