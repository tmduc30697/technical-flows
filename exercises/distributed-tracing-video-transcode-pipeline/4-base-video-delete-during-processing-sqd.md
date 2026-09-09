# Sequence - Base: Xoá video giữa lúc đang xử lý (trace treo mãi mãi)

Đây là flow **base**: người dùng xoá video trong lúc job thumbnail vẫn đang nằm trong hàng đợi chờ worker rảnh. Job này (và trace của nó) không được ai chủ động đóng lại, nên trên dashboard giám sát nó hiển thị mãi ở trạng thái "đang xử lý" dù thực tế video đã bị xoá và sẽ không bao giờ hoàn tất. Đây là tiền đề cho yêu cầu 5 của đề bài.

```mermaid
sequenceDiagram
    participant User as Người dùng
    participant API as Video API
    participant Queue as Job Queue
    participant Thumb as Worker Thumbnail
    participant Dash as Dashboard giám sát

    Queue->>Queue: Job tạo thumbnail đang chờ trong hàng đợi
    Note over Queue: Trace #5 đã được tạo, span ở trạng thái "running"
    User->>API: Xoá video
    API->>API: Đánh dấu video status=deleted trong DB
    API-->>User: Xoá thành công
    Note over Queue: Job tạo thumbnail vẫn còn trong hàng đợi, không ai huỷ nó
    Queue->>Thumb: Cuối cùng worker rảnh, lấy job ra chạy
    Thumb->>Thumb: Đọc video từ storage, phát hiện video không còn tồn tại
    Thumb-->>Queue: Job thất bại âm thầm hoặc bị bỏ qua
    Note over Queue: Span của trace #5 không bao giờ được đóng lại rõ ràng
    Dash->>Dash: Vẫn hiển thị trace #5 ở trạng thái "đang xử lý" mãi mãi
```
