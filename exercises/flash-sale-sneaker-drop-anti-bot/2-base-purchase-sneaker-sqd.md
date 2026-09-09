# Base sequence — Purchase sneaker (mua trực tiếp, không chống race, không chống bot)

Đây là **base**, flow "Mua giày lúc mở bán" ở trạng thái hiện tại — kiểm tra tồn kho rồi trừ tồn kho bằng 2 bước riêng biệt (đọc rồi ghi), không có ràng buộc DB nào ngăn 1 user mua nhiều lần, không có captcha, không có idempotency key. Flow này liên quan mật thiết tới enhance vì cả 5 yêu cầu của đề bài đều là các lớp bảo vệ được thêm vào đúng flow này.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Checkout Service
    participant DB as EVENT/ORDER store

    Customer->>App: Bấm mua (event_id, user_id, shipping_address)
    App->>DB: SELECT remaining_stock FROM EVENT WHERE id=event_id
    DB-->>App: remaining_stock = 1
    Note over App,DB: Một request khác của cùng user (tab thứ 2) cũng vừa đọc remaining_stock = 1 tại cùng thời điểm
    App->>DB: INSERT INTO ORDER (user_id, event_id, status=pending)
    App->>DB: UPDATE EVENT SET remaining_stock = remaining_stock - 1
    DB-->>App: OK
    App-->>Customer: "Đặt hàng thành công"
    Note over App,DB: Không có unique constraint (event_id, user_id) nên cùng 1 user có thể tạo được 2 ORDER, tồn kho có thể bị trừ âm hoặc bán vượt 500 cặp khi nhiều request đọc-ghi chồng chéo
```
