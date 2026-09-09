# Base sequence — View balance (không có fallback khi core chậm)

Đây là **base**, flow "Xem số dư" — gọi thẳng core banking real-time, không có phương án dự phòng khi core chậm/quá tải. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 3 của đề bài chính là thêm fallback rõ ràng ở đây.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Digital Banking App
    participant Core as Core Banking (legacy)

    Customer->>App: Xem số dư tài khoản
    App->>Core: Gọi lấy số dư real-time
    alt Core phản hồi bình thường
        Core-->>App: Trả số dư
        App-->>Customer: Hiển thị số dư
    else Core đang chậm/quá tải (cao điểm)
        Core-->>App: Timeout/lỗi
        App-->>Customer: "Không thể tải số dư, vui lòng thử lại sau"
        Note over App,Core: Không có cache dự phòng nào để hiển thị tạm, trải nghiệm hoàn toàn gián đoạn dù chỉ là thao tác đọc ít rủi ro
    end
```
