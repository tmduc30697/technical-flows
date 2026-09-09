# Base sequence — View shop data

Đây là **base**, flow "Chủ shop xem dữ liệu order/customer/inventory qua trang quản trị" — chọn flow này để làm rõ mô hình truy cập dữ liệu hiện tại: chỉ qua session của chính chủ shop, chưa có khái niệm 1 bên thứ ba nào khác gọi vào cùng API này. Đây là baseline mà flow enhance "app thứ ba gọi API" (Bước 6) sẽ mở rộng thêm 1 đường truy cập hoàn toàn mới bên cạnh, chứ không thay thế flow này.

```mermaid
sequenceDiagram
    actor Owner as Shop Owner
    participant AdminDashboard
    participant API as Resource API
    participant DB as Database

    Owner->>AdminDashboard: Mở trang Orders
    AdminDashboard->>API: GET /api/orders (session_token)
    API->>DB: Xác thực session, xác định shop_id của owner
    DB-->>API: user hợp lệ, shop_id
    API->>DB: SELECT orders WHERE shop_id = :shop_id
    DB-->>API: danh sách order
    API-->>AdminDashboard: 200 danh sách order
    AdminDashboard-->>Owner: Hiển thị danh sách order
    Note over API,DB: Chưa có khái niệm app bên ngoài, client_id, hay access token có scope
```
