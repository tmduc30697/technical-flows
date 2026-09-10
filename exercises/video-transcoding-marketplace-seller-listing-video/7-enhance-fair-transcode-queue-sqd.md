# Sequence Diagram — Enhance: Fair Transcode Queue

Đây là **enhance**, flow mới xử lý yêu cầu công bằng hàng đợi giữa các seller ở giờ cao điểm: tránh một seller upload nhiều video liên tục chiếm hết tài nguyên khiến seller khác chờ lâu bất thường, đồng thời kiểm soát chi phí xử lý khi khối lượng nhỏ lẻ nhưng số lượng lớn.

```mermaid
sequenceDiagram
    participant Marketplace as Marketplace Service
    participant Queue as Video Processing Queue
    participant Quota as Seller Queue Quota
    participant Worker as Transcode Worker

    Marketplace->>Queue: Enqueue VIDEO_PROCESSING_JOB (seller_id)
    Queue->>Quota: Check SELLER_QUEUE_QUOTA cho seller này

    alt seller đã có nhiều job đang chờ/chạy vượt ngưỡng
        Quota-->>Queue: concurrent_jobs vượt giới hạn
        Queue->>Queue: Đẩy job này xuống vị trí sau trong hàng đợi công bằng (round-robin theo seller)
    else trong ngưỡng cho phép
        Quota-->>Queue: OK
        Queue->>Worker: Dispatch job theo thứ tự công bằng giữa các seller
    end

    Worker-->>Queue: Job completed
    Queue->>Quota: Giảm concurrent_jobs của seller
```
