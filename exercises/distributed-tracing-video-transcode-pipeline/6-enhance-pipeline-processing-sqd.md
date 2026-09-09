# Sequence - Enhance: Xử lý pipeline video (1 trace gốc, span song song đúng thực tế)

Đây là flow **enhance** của `pipeline-processing` (so với base). Khác biệt so với base: toàn bộ job của video (kiểm duyệt, 3 job transcode, thumbnail, publish) đều sinh span dưới cùng 1 `trace_id` gốc gắn với video, và 3 span transcode song song đều là con trực tiếp của 1 span cha "transcode", nên trace thể hiện đúng chúng chạy song song thay vì bị hiểu lầm là tuần tự. Đáp ứng **yêu cầu 1 và 2** của đề bài.

```mermaid
sequenceDiagram
    participant User as Người dùng
    participant API as Upload API
    participant Queue as Job Queue
    participant Mod as Worker Kiểm duyệt
    participant T1 as Worker Transcode 480p
    participant T2 as Worker Transcode 720p
    participant T3 as Worker Transcode 1080p
    participant Thumb as Worker Thumbnail
    participant Pub as Worker Publish

    User->>API: Upload video
    API->>API: Tạo trace gốc (trace_id) gắn với video_id
    API->>Queue: Enqueue job kiểm duyệt (kèm trace_id gốc)
    Queue->>Mod: Chạy job kiểm duyệt
    Note over Mod: Span con của trace gốc, job_type=moderation
    Mod-->>Queue: Kiểm duyệt đạt

    Note over Queue: Tạo 1 span cha "transcode" trong trace gốc, 3 job dưới đây là con song song của span cha này
    par Transcode song song 3 độ phân giải
        Queue->>T1: Enqueue job transcode 480p (cùng trace_id, parent=span cha transcode)
        T1-->>Queue: Xong 480p
    and
        Queue->>T2: Enqueue job transcode 720p (cùng trace_id, parent=span cha transcode)
        T2-->>Queue: Xong 720p
    and
        Queue->>T3: Enqueue job transcode 1080p (cùng trace_id, parent=span cha transcode)
        T3-->>Queue: Xong 1080p
    end
    Note over Queue: Dashboard thấy rõ 3 span sibling có start_time trùng nhau, đúng là chạy song song

    Queue->>Thumb: Enqueue job tạo thumbnail (cùng trace_id gốc)
    Thumb-->>Queue: Xong thumbnail

    Queue->>Pub: Enqueue job publish (cùng trace_id gốc)
    Pub-->>Queue: Video đã publish
    Queue-->>API: Pipeline hoàn tất, đóng trace gốc
```
