# Base sequence — Fallback status polling

Đây là **base**, flow "Polling dự phòng khi callback chưa tới" — đề bài nhắc tới flow này như bối cảnh có sẵn ("do quá lâu chưa nhận được callback nên có luồng dự phòng tự chuyển trạng thái theo polling"). Chọn dựng flow này vì hệ quả của nó (order bị chuyển tạm sang "đã thanh toán" trước khi có callback chính thức) chính là tình huống enhance phải xử lý đảo ngược đúng cách khi callback thật đến sau đó báo thất bại.

```mermaid
sequenceDiagram
    participant Scheduler
    participant OrderService
    participant DB as Database
    participant PaymentPartner

    Scheduler->>OrderService: Kích hoạt job kiểm tra order pending quá lâu chưa có callback
    OrderService->>DB: Lấy các ORDER đang pending quá ngưỡng thời gian
    OrderService->>PaymentPartner: GET /payments/{partner_ref}/status
    PaymentPartner-->>OrderService: status=success (theo polling)
    OrderService->>DB: Cập nhật ORDER status=paid tạm thời dựa trên kết quả polling
    Note over OrderService,DB: Đây là trạng thái tạm, chưa phải callback chính thức từ đối tác
    OrderService-->>Scheduler: Job hoàn tất
    Note over OrderService,DB: Base chưa có cơ chế xử lý đúng nếu sau đó callback chính thức báo thất bại
```
