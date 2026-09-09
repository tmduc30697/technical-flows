# Enhance sequence — Archive partition cũ sang cold storage

Đây là **enhance**, flow hoàn toàn mới, chưa tồn tại ở base (base không có cơ chế archive). Đáp ứng yêu cầu 3 của đề bài: partition cũ hơn X ngày được tự động nén/di chuyển sang storage rẻ hơn mà không làm gián đoạn ghi dữ liệu mới của các partition khác (kể cả partition đang active).

```mermaid
sequenceDiagram
    participant Scheduler as Archive Scheduler
    participant PartOld as PARTITION (time_bucket đã quá X ngày)
    participant Job as ARCHIVE_JOB
    participant Cold as Cold Storage
    participant PartNew as PARTITION (time_bucket hiện tại, đang nhận ghi)

    Scheduler->>PartOld: Quét định kỳ, tìm partition có time_bucket > X ngày tuổi
    Scheduler->>Job: Tạo ARCHIVE_JOB (status=pending) cho partition cũ
    Job->>Job: Chuyển status=archiving
    Job->>PartOld: Đọc snapshot dữ liệu (read-only, không lock ghi)
    Job->>Job: Nén dữ liệu
    Job->>Cold: Ghi object đã nén lên cold storage
    Cold-->>Job: OK, trả về cold_storage_uri
    Job->>Job: Chuyển status=done, cập nhật archived_at

    par Song song, không bị gián đoạn
        PartNew->>PartNew: Vẫn tiếp tục nhận ghi metric point bình thường
    end
    Note over Job,PartOld: Sau khi archive xong, partition cũ có thể được giải phóng khỏi node nóng, dữ liệu vẫn truy vấn được (chậm hơn) từ cold storage khi cần
```
