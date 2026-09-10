# Sequence Diagram — Enhance: Autoscale and Backpressure

Đây là **enhance**, flow mới hoàn toàn phục vụ yêu cầu chịu tải khi upload tăng đột biến (viral moment, giờ cao điểm): tự động scale worker theo độ sâu hàng đợi và có cơ chế backpressure để không sập toàn hệ thống.

```mermaid
sequenceDiagram
    participant PriorityQueue as Priority Transcode Queue
    participant Monitor as Queue Monitor
    participant Pool as Worker Pool
    participant VideoSvc as Video Service

    Monitor->>PriorityQueue: Theo dõi queue_depth liên tục
    alt queue_depth tăng vượt ngưỡng
        Monitor->>Pool: Yêu cầu scale up active_workers
        Pool->>Pool: Khởi tạo thêm worker
        Pool-->>Monitor: Đã scale, queue_depth giảm dần
    else queue_depth vượt ngưỡng chịu đựng tối đa kể cả sau scale
        Monitor->>Pool: Set backpressure_state = active
        Pool-->>VideoSvc: Báo backpressure
        VideoSvc-->>VideoSvc: Tạm hoãn nhận upload mới hoặc hạ độ ưu tiên non-critical
        Note over VideoSvc,Pool: Bảo vệ hệ thống khỏi sập thay vì cố nhận hết tải
    end

    Monitor->>Pool: queue_depth trở lại bình thường, scale down / tắt backpressure
```
