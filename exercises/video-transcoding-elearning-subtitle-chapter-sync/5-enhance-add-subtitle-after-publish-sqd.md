# Sequence Diagram — Enhance: Add Subtitle After Publish

Đây là **enhance**, flow hoàn toàn mới: giảng viên upload phụ đề sau khi video đã transcode xong và publish, việc gắn thêm phụ đề chỉ chạy pipeline phụ đề độc lập, không yêu cầu transcode lại toàn bộ video hay các RENDITION đã có sẵn.

```mermaid
sequenceDiagram
    actor Instructor as Giảng viên
    participant Platform as Nền tảng e-learning
    participant SubtitlePipeline as Subtitle/Chapter Sync Pipeline

    Instructor->>Platform: Upload file phụ đề mới cho video đã publish
    Platform->>Platform: Kiểm tra video đã có TRANSCODE_JOB hoàn tất với danh sách TRANSFORM_STEP
    Platform->>SubtitlePipeline: Yêu cầu đồng bộ timestamp cho phụ đề mới, không đụng tới RENDITION

    SubtitlePipeline->>SubtitlePipeline: Áp dụng lại toàn bộ TRANSFORM_STEP đã ghi nhận để map timestamp phụ đề mới
    SubtitlePipeline-->>Platform: SUBTITLE_VERSION mới đã đồng bộ đúng mốc video hiện có

    Platform->>Platform: Tạo SUBTITLE_TRACK/SUBTITLE_VERSION mới, is_current=true
    Platform-->>Instructor: Phụ đề mới đã sẵn sàng, video và các RENDITION không bị ảnh hưởng
```
