# Sequence Diagram - Enhance: fraud-check-append-only

Đây là flow **enhance mới**: span của bước kiểm tra fraud được ghi kèm `integrity_hash` và đánh dấu `is_append_only=true`; khi có yêu cầu sửa/ghi đè lại kết quả span này (dù vô tình hay cố ý), hệ thống từ chối và chỉ cho phép ghi thêm 1 span/bút toán điều chỉnh mới, không xoá/sửa bản gốc - tránh tranh chấp "hệ thống đã check fraud hay chưa". Đáp ứng yêu cầu 2.

```mermaid
sequenceDiagram
    participant Fraud as fraud-check
    participant Store as Trace Storage (append-only)
    actor Engineer as Kỹ sư vận hành

    Fraud->>Store: Ghi span S2, kết quả "PASS", is_append_only=true, integrity_hash=H1
    Store->>Store: Lưu span S2, khoá không cho sửa/ghi đè
    Store-->>Fraud: Ghi thành công
    Note over Engineer: Sau đó phát hiện cần điều chỉnh do lỗi phát hiện muộn
    Engineer->>Store: Yêu cầu sửa trực tiếp span S2 thành "FLAGGED"
    Store->>Store: Kiểm tra is_append_only=true trên span S2
    Store-->>Engineer: Từ chối sửa, span gốc không được phép chỉnh sửa
    Engineer->>Store: Ghi thêm span mới S2b "correction", tham chiếu ngược span S2, kết quả "FLAGGED"
    Store->>Store: Lưu span S2b như 1 bản ghi mới, không thay thế S2
    Store-->>Engineer: Ghi thành công, lịch sử đầy đủ cả S2 gốc và S2b điều chỉnh vẫn còn nguyên vẹn
```
