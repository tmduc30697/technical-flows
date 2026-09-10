# Sequence Diagram — Enhance: End Meeting and Process Recording

Đây là **enhance**, flow đã thay đổi so với base ([2-base-end-meeting-process-recording-sqd.md](2-base-end-meeting-process-recording-sqd.md)): thay vì vào hàng đợi chuẩn dùng chung, recording nay được xử lý trên priority queue riêng, tài nguyên compute ưu tiên cao hơn (chấp nhận chi phí cao hơn), và job/storage được cô lập theo `org_id` ngay từ đầu.

```mermaid
sequenceDiagram
    participant MeetingSvc as Meeting Service
    participant Storage as Recording Storage (per-org isolated)
    participant PriorityQueue as Priority Transcode Queue
    participant Worker as Dedicated Fast Worker

    MeetingSvc->>MeetingSvc: Meeting ended
    MeetingSvc->>Storage: Store RECORDING vào namespace riêng của org
    MeetingSvc->>PriorityQueue: Enqueue PROCESSING_JOB, priority_tier = high
    Note over PriorityQueue: Job này chen trước hàng đợi thông thường, chấp nhận compute cost cao hơn

    PriorityQueue->>Worker: Dispatch gần như ngay lập tức
    Worker->>Storage: Fetch RECORDING (chỉ đọc trong namespace của org đó)
    Worker->>Storage: Transcode ưu tiên tốc độ
    Worker-->>MeetingSvc: Recording ready trong thời gian ngắn
    MeetingSvc-->>MeetingSvc: Gửi bản ghi cho khách hàng khi còn "nóng"
```
