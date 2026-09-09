# ERD - Base: Đăng ký vay ưu đãi với suất còn lại đọc-rồi-ghi

Đây là trạng thái **base**, trước khi áp đề bài. App ngân hàng số đã có chương trình vay ưu đãi giới hạn số suất, khách hàng bấm đăng ký, backend kiểm tra `remaining_slots` rồi trừ đi 1 (đọc-rồi-ghi, không atomic), sau đó gọi kiểm tra tín dụng đồng bộ (block chờ vài giây). Không có khái niệm "vị trí hàng đợi" tách biệt khỏi việc cấp suất, nên suất bị trừ ngay tại thời điểm bấm đăng ký chứ không phải sau khi qua được kiểm tra tín dụng. Đây là nền tảng để đề bài (giữ vị trí atomic, nhả suất cho người kế tiếp, atomic counter, retry/timeout, dashboard real-time) có ý nghĩa khi so sánh.

```mermaid
erDiagram
    LOAN_PROGRAM ||--o{ REGISTRATION : receives
    CUSTOMER ||--o{ REGISTRATION : submits
    REGISTRATION ||--o| CREDIT_CHECK : "kiểm tra đồng bộ"

    LOAN_PROGRAM {
        string program_id
        int total_slots
        int remaining_slots
        datetime start_time
    }
    CUSTOMER {
        string customer_id
        string name
        string national_id
    }
    REGISTRATION {
        string registration_id
        string program_id
        string customer_id
        string status
        datetime created_at
    }
    CREDIT_CHECK {
        string check_id
        string registration_id
        string result
        datetime checked_at
    }
```

Lưu ý: `LOAN_PROGRAM.remaining_slots` được cập nhật bằng đọc giá trị hiện tại rồi ghi giá trị mới (không atomic), `REGISTRATION` không có field vị trí hàng đợi (`queue_position`) tách biệt, và `CREDIT_CHECK` chỉ có 1 lần kiểm tra duy nhất, không có `attempt_number` hay cơ chế timeout/retry — đây chính là các khoảng trống mà enhance sẽ lấp.
