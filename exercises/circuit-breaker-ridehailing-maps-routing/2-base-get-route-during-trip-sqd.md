# Base sequence — Get route during trip (treo màn hình, retry kiểu batch)

Đây là **base**, flow "Lấy route/ETA trong lúc chuyến đi đang chạy" ở trạng thái hiện tại — coi như mọi API khác, retry với backoff dài, không có fallback nào trong lúc chờ. Flow này liên quan mật thiết tới enhance vì toàn bộ 5 yêu cầu của đề bài đều nhằm sửa đúng các lỗ hổng nguy hiểm ở đây.

```mermaid
sequenceDiagram
    actor Driver
    participant App as Driver App
    participant Maps as Maps API (bên thứ ba)

    Note over Driver: Xe đang di chuyển thực tế trong chuyến đi
    App->>Maps: Gọi tính route/ETA
    Maps-->>App: Chậm, mất vài giây mới phản hồi
    Note over App,Driver: Màn hình chỉ đường bị treo/trắng trong lúc chờ, tài xế mất định hướng khi đang lái xe
    Maps-->>App: Cuối cùng timeout
    App->>Maps: Retry với backoff dài (giống hệt cách xử lý cho luồng batch khác)
    Maps-->>App: Trả route thành công
    Note over App,Driver: Nhưng lúc này xe đã di chuyển tiếp, route trả về đã lỗi thời so với vị trí hiện tại
    App-->>Driver: Hiển thị route (đã cũ)
```
