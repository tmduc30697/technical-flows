# Sequence Diagram — Enhance: Upload and Transcode Video

Đây là **enhance**, flow đã thay đổi so với base ([2-base-upload-and-transcode-video-sqd.md](2-base-upload-and-transcode-video-sqd.md)): video nay đi qua kiểm tra sơ bộ trước khi vào hàng đợi riêng ưu tiên SLA, và được chuẩn hóa nhiều tỉ lệ khung hình/framerate khác nhau về cùng chuẩn output thay vì transcode 1 định dạng chung như base.

```mermaid
sequenceDiagram
    actor User
    participant App as Mobile App
    participant VideoSvc as Video Service
    participant ValidationSvc as Pre-check Service
    participant PriorityQueue as Priority Transcode Queue
    participant Worker as Transcode Worker

    User->>App: Upload video
    App->>VideoSvc: Submit video
    VideoSvc->>ValidationSvc: Check duration, size, định dạng hợp lệ

    alt không hợp lệ
        ValidationSvc-->>VideoSvc: passed = false, reason
        VideoSvc-->>User: Từ chối sớm, không vào hàng đợi
    else hợp lệ
        ValidationSvc-->>VideoSvc: passed = true
        VideoSvc->>PriorityQueue: Enqueue TRANSCODE_JOB, priority_queue = short-video-sla
        PriorityQueue->>Worker: Dispatch nhanh, mục tiêu vài giây đến vài chục giây
        Worker->>Worker: Chuẩn hóa aspect_ratio (9:16, 1:1, 16:9) và framerate về output thống nhất
        Worker-->>VideoSvc: Ít nhất 1 rendition sẵn sàng
        VideoSvc-->>User: Video "có thể xem được" trong SLA
    end
```
