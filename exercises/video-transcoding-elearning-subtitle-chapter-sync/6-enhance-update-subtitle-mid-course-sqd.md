# Sequence Diagram — Enhance: Update Subtitle Mid Course

Đây là **enhance**, flow hoàn toàn mới xử lý khi giảng viên chỉnh sửa phụ đề sau khi khóa học đã có học viên đang học — bản phụ đề cũ không bị thay đổi đột ngột giữa chừng nhờ versioning, bản mới chỉ áp dụng từ lần xem tiếp theo của học viên.

```mermaid
sequenceDiagram
    actor Instructor as Giảng viên
    participant Platform as Nền tảng e-learning
    actor Student as Học viên đang xem dở

    Instructor->>Platform: Chỉnh sửa nội dung phụ đề cho video đang có học viên học
    Platform->>Platform: Tạo SUBTITLE_VERSION mới (version_number+1), is_current=false tạm thời

    Note over Student,Platform: Học viên đang có STUDENT_VIEW_SESSION pin vào SUBTITLE_VERSION cũ, không bị đổi giữa chừng
    Student->>Platform: Tiếp tục xem video, dùng SUBTITLE_VERSION đã pin từ đầu session

    Platform->>Platform: Đánh dấu SUBTITLE_VERSION mới is_current=true cho các session mới
    Student->>Platform: Học viên mở lại video ở lần xem tiếp theo, tạo STUDENT_VIEW_SESSION mới
    Platform->>Platform: Pin session mới vào SUBTITLE_VERSION hiện hành (bản đã sửa)
    Platform-->>Student: Hiển thị bản phụ đề mới từ lần xem này trở đi
```
