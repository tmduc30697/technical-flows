# Enhance sequence — Retry tự động đọc lại tồn kho hiện tại, không dùng số liệu cũ

Đây là **enhance**, flow mới bổ sung cho trường hợp deadlock vẫn xảy ra dù đã chuẩn hoá thứ tự lock (ví dụ 3 phiếu chồng lấn SKU cùng lúc tạo tranh chấp ngoài dự kiến). Đáp ứng yêu cầu 3 của đề bài: khi bị rollback do deadlock, transaction phải retry tự động nhưng bắt buộc đọc lại tồn kho hiện tại tại thời điểm retry, không tái sử dụng số liệu đã đọc ở lần thử trước, vì trong lúc chờ retry có thể đã có phiếu khác làm thay đổi tồn kho các dòng liên quan.

```mermaid
sequenceDiagram
    actor NV as Nhân viên (P1)
    participant App as Transfer Service
    participant DB as Database
    participant InvB102 as INVENTORY (B,102)
    participant OtherOrder as Phiếu P3 khác (không liên quan trực tiếp NV)
    participant Log as DEADLOCK_LOG

    NV->>App: Gửi phiếu chuyển P1 (A→B, SKU101+SKU102)
    App->>DB: BEGIN P1 (retry_attempt=1), đọc tồn kho hiện tại các dòng liên quan
    DB-->>App: quantity(B,102) = 50 tại thời điểm đọc lần 1
    DB-->>App: Lỗi deadlock (mã 40P01/1213) khi lock, transaction rollback tự động
    App->>Log: Ghi DEADLOCK_LOG (transfer_order_id=P1, locked_resource=(B,102), db_error_code, retry_attempt=1, occurred_at)

    Note over OtherOrder,InvB102: Trong lúc App đang backoff, phiếu P3 khác commit thành công, làm quantity(B,102) đổi từ 50 xuống 30

    App->>App: Backoff (vd 100ms) rồi chuẩn bị retry
    App->>DB: BEGIN P1 (retry_attempt=2), ĐỌC LẠI tồn kho hiện tại, không dùng số liệu quantity=50 đã đọc trước đó
    DB-->>App: quantity(B,102) = 30, giá trị mới nhất
    App->>App: Tính lại số lượng khả dụng để chuyển dựa trên quantity=30 vừa đọc
    App->>DB: Lock toàn bộ dòng theo composite key, trừ/cộng theo số liệu mới, COMMIT
    DB-->>App: Transaction P1 thành công

    App-->>NV: Chuyển hàng thành công, nhân viên không hề thấy lỗi deadlock trung gian
    Note over App,Log: Nếu retry đọc lại vẫn dùng số liệu cũ quantity=50, phiếu có thể trừ vượt quá tồn kho thực tế 30, gây âm kho
```
