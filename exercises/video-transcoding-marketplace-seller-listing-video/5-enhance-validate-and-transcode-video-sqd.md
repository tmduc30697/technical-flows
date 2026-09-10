# Sequence Diagram — Enhance: Validate and Transcode Video

Đây là **enhance**, flow mới xử lý một `VIDEO_PROCESSING_JOB`: nhận diện đa dạng codec/độ phân giải/tỉ lệ khung hình/hướng quay, kiểm tra ngưỡng chất lượng tối thiểu để từ chối sớm, rồi mới transcode ra output chuẩn hóa — tránh lãng phí tài nguyên cho video chất lượng quá thấp không thể cải thiện.

```mermaid
sequenceDiagram
    participant Queue as Video Processing Queue
    participant Worker as Transcode Worker
    participant QualitySvc as Quality Check Service
    participant Storage as Media Storage
    participant Marketplace as Marketplace Service

    Queue->>Worker: Dispatch VIDEO_PROCESSING_JOB
    Worker->>Storage: Fetch VIDEO_RAW (codec, resolution, aspect_ratio, orientation)
    Worker->>QualitySvc: Run VIDEO_QUALITY_CHECK (sharpness, brightness)

    alt chất lượng dưới ngưỡng tối thiểu
        QualitySvc-->>Worker: passed = false, reason
        Worker->>Marketplace: Reject job, cảnh báo seller kèm lý do
        Marketplace-->>Worker: Acknowledged
    else đạt ngưỡng
        QualitySvc-->>Worker: passed = true
        Worker->>Worker: Detect input codec/resolution/aspect_ratio/orientation
        Worker->>Worker: Normalize về output chuẩn thống nhất (tự động xoay, crop-safe)
        Worker->>Storage: Store VIDEO_RENDITION
        Worker-->>Queue: Job completed
    end
```
