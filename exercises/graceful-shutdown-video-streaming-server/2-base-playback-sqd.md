# Sequence - Base - Flow "playback"

Đây là **base**: flow phát video bình thường, khi chưa có bất kỳ cơ chế shutdown nào. Flow này được chọn làm nền vì nó xác lập bản chất của kết nối streaming - một request duy nhất nhưng giữ mở rất lâu, đọc liên tục nhiều chunk - chính là lý do khiến shutdown thông thường (đóng kết nối sau vài chục giây) không áp dụng được.

```mermaid
sequenceDiagram
    actor Client
    participant LB as Load Balancer
    participant Instance
    participant Storage as Video Storage

    Client->>LB: GET /stream/video-123
    LB->>Instance: forward request (chọn instance còn tải nhẹ)
    Instance->>Storage: mở stream video-123
    Instance-->>Client: 200 OK, bắt đầu gửi chunk video

    loop Trong suốt thời lượng video
        Instance->>Storage: đọc chunk tiếp theo
        Instance-->>Client: gửi chunk qua kết nối đang mở
    end

    alt Video phát hết tự nhiên
        Instance-->>Client: đóng kết nối, kết thúc bình thường
    else Người dùng chủ động dừng xem
        Client->>Instance: đóng kết nối
        Instance-->>Instance: dọn session, log kết thúc bởi người dùng
    end
```
