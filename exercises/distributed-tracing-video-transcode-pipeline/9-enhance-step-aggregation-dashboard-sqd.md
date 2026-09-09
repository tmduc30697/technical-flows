# Sequence - Enhance: Tổng hợp latency theo từng bước trên toàn hệ thống

Đây là flow **enhance hoàn toàn mới**, không tồn tại ở base (base chỉ có trace riêng lẻ theo từng job/video, không có tầng tổng hợp theo bước trên toàn hệ thống). Flow này mô tả hệ thống gom span theo `job_type` (vd transcode 1080p) từ mọi video, mọi trace, để trả lời câu hỏi "bước transcode 1080p có đang chậm dần theo thời gian do thiếu tài nguyên worker hay không". Đáp ứng **yêu cầu 4** của đề bài.

```mermaid
sequenceDiagram
    participant Tracer as Tracing Backend
    participant Agg as Metrics Aggregator
    participant Store as STEP_METRIC Store
    participant Ops as Đội vận hành

    loop Mỗi khoảng thời gian (vd 5 phút)
        Tracer->>Agg: Đẩy toàn bộ span đã đóng (job_type, duration_ms, status) từ mọi trace, mọi video
        Agg->>Agg: Gom nhóm theo job_type=transcode-1080p, tính p95 duration_ms và error_rate trong khung 5 phút
        Agg->>Store: Ghi STEP_METRIC(job_type=transcode-1080p, time_bucket, p95_duration_ms, error_rate)
    end

    Ops->>Store: Truy vấn xu hướng p95_duration_ms của transcode-1080p theo thời gian
    Store-->>Ops: p95 tăng dần liên tục qua các time_bucket gần đây
    Note over Ops: Xu hướng tăng dần đều trên toàn hệ thống (không phải 1 video cụ thể) gợi ý nguyên nhân là thiếu tài nguyên worker pool transcode-1080p, không phải lỗi ở 1 video đơn lẻ
    Ops->>Ops: Quyết định mở rộng thêm worker cho pool transcode-1080p
```
