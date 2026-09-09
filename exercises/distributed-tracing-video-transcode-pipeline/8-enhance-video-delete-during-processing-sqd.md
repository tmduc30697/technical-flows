# Sequence - Enhance: Xoá video giữa lúc đang xử lý (đóng span với trạng thái cancelled)

Đây là flow **enhance** của `video-delete-during-processing` (so với base). Khác biệt so với base: khi video bị xoá, API chủ động tìm mọi span chưa hoàn tất thuộc `trace_id` gốc của video đó (bao gồm cả job còn đang nằm trong hàng đợi) và đóng lại rõ ràng với `status=cancelled`, thay vì để trace treo mãi ở trạng thái "đang xử lý". Đáp ứng **yêu cầu 5** của đề bài.

```mermaid
sequenceDiagram
    participant User as Người dùng
    participant API as Video API
    participant Queue as Job Queue
    participant Thumb as Worker Thumbnail
    participant Tracer as Tracing Backend
    participant Dash as Dashboard giám sát

    Queue->>Queue: Job tạo thumbnail đang chờ trong hàng đợi
    Note over Queue: Span con trong trace gốc, status=running
    User->>API: Xoá video
    API->>API: Đánh dấu video status=deleted, deleted_at=now()
    API->>Queue: Huỷ mọi job chưa chạy của video này (job tạo thumbnail, publish nếu có)
    API->>Tracer: Đóng mọi span chưa hoàn tất của trace_id gốc với status=cancelled
    API-->>User: Xoá thành công
    Note over Tracer: Span "tạo thumbnail" và các span con khác chuyển ngay sang cancelled, không còn ở trạng thái running
    Queue-->>Thumb: (Job đã bị huỷ, worker không cần xử lý nữa)
    Dash->>Dash: Hiển thị trace của video là "đã huỷ", không còn treo ở "đang xử lý"
```
