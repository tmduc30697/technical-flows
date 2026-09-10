# Sequence Diagram — Enhance: Cancel Transcode on Delete

Đây là **enhance**, flow mới xử lý trường hợp user xóa video trong lúc đang transcode — hủy job giữa chừng và dọn sạch file trung gian, không tiếp tục xử lý lãng phí tài nguyên worker.

```mermaid
sequenceDiagram
    actor User
    participant VideoSvc as Video Service
    participant PriorityQueue as Priority Transcode Queue
    participant Worker as Transcode Worker
    participant Storage as Media Storage

    User->>VideoSvc: Xóa video (đang transcode)
    VideoSvc->>VideoSvc: Set VIDEO.status = deleted
    VideoSvc->>PriorityQueue: Yêu cầu hủy TRANSCODE_JOB tương ứng

    alt job chưa được worker nhận
        PriorityQueue->>PriorityQueue: Remove job khỏi hàng đợi trước khi dispatch
    else job đang chạy trên worker
        PriorityQueue->>Worker: Signal cancel
        Worker->>Worker: Dừng xử lý giữa chừng
        Worker->>Storage: Xóa file trung gian/rendition dở dang
    end

    Worker-->>VideoSvc: Job status = cancelled
    VideoSvc-->>User: Video đã xóa hoàn toàn, không còn job chạy nền
```
