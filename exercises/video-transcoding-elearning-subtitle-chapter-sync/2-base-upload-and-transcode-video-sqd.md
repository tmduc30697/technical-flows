# Sequence Diagram — Base: Upload And Transcode Video

Đây là **base**, flow giảng viên upload video, hệ thống transcode 1 bước ra nhiều độ phân giải, phụ đề/chapter marker được gắn thẳng theo timestamp gốc mà không tính toán lại — đây chính là quy trình sẽ được thay đổi ở enhance vì mốc thời gian bị lệch khi transcode làm thay đổi độ dài video.

```mermaid
sequenceDiagram
    actor Instructor as Giảng viên
    participant Platform as Nền tảng e-learning
    participant Transcoder as Transcode Service

    Instructor->>Platform: Upload video bài giảng
    Instructor->>Platform: Đánh dấu CHAPTER_MARKER thủ công trên bản gốc
    Instructor->>Platform: Upload SUBTITLE_FILE theo timestamp bản gốc

    Platform->>Transcoder: Transcode video (1 bước, ra nhiều RENDITION)
    Transcoder-->>Platform: Hoàn tất, danh sách RENDITION

    Platform->>Platform: Gắn thẳng SUBTITLE_FILE và CHAPTER_MARKER theo timestamp gốc, không tính lại
    Platform-->>Instructor: Video đã publish cho học viên
    Note over Platform: Nếu transcode làm thay đổi độ dài video, timestamp phụ đề/chapter sẽ bị lệch dần
```
