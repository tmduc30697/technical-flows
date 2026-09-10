# Sequence Diagram — Enhance: Upload And Transcode Video

Đây là **enhance**, flow transcode thay đổi cốt lõi so với base: pipeline nay chạy nhiều `TRANSFORM_STEP` (cắt intro, chuẩn hóa framerate, đổi tốc độ), mỗi bước ghi nhận offset/scale để tính lại timestamp chapter marker chính xác, đồng thời pipeline phụ đề/chapter chạy song song độc lập nên video có thể sẵn sàng cho học viên trước, không phải chờ phụ đề xử lý xong như một khối với base.

```mermaid
sequenceDiagram
    actor Instructor as Giảng viên
    participant Platform as Nền tảng e-learning
    participant Transcoder as Transcode Service
    participant SubtitlePipeline as Subtitle/Chapter Sync Pipeline

    Instructor->>Platform: Upload video bài giảng
    Instructor->>Platform: Đánh dấu CHAPTER_MARKER thủ công (original_timestamp_seconds)

    Platform->>Transcoder: Transcode video, chạy tuần tự nhiều TRANSFORM_STEP
    Transcoder->>Transcoder: Bước 1 - cắt intro thừa (ghi offset_seconds)
    Transcoder->>Transcoder: Bước 2 - chuẩn hóa framerate
    Transcoder->>Transcoder: Bước 3 - đổi tốc độ phát nếu có (ghi scale_factor)
    Transcoder-->>Platform: Hoàn tất RENDITION, transcoded_duration_seconds

    par pipeline video chính
        Platform-->>Instructor: Video sẵn sàng cho học viên xem ngay
    and pipeline phụ đề/chapter chạy song song độc lập
        Platform->>SubtitlePipeline: Tính lại transcoded_timestamp_seconds cho từng CHAPTER_MARKER dựa trên toàn bộ TRANSFORM_STEP
        SubtitlePipeline->>SubtitlePipeline: Áp dụng offset/scale của từng bước theo đúng thứ tự
        SubtitlePipeline-->>Platform: Cập nhật CHAPTER_MARKER, tạo SUBTITLE_VERSION đầu tiên
        Platform-->>Instructor: Phụ đề/chapter cập nhật bổ sung, có thể muộn hơn thời điểm video sẵn sàng
    end
```
