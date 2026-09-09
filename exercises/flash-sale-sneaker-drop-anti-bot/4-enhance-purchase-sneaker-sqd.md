# Enhance sequence — Purchase sneaker (challenge trước hàng đợi, transaction atomic độc lập)

Đây là **enhance** của flow `purchase-sneaker` đã có ở base, đáp ứng **yêu cầu 1 và 2** của đề bài. So với base: (1) trước khi vào hàng đợi mua, request phải vượt qua `CHALLENGE_TOKEN` (captcha), (2) việc trừ tồn kho và tạo đơn hàng nay là 1 transaction DB atomic dùng ràng buộc `UNIQUE(event_id, user_id)` cộng điều kiện `WHERE remaining_stock > 0`, độc lập hoàn toàn với lớp challenge — tức là kể cả nếu challenge bị bot vượt qua được, transaction vẫn tự bảo vệ đúng đắn tồn kho và không cho mua trùng.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Checkout Service
    participant DB as EVENT/ORDER store

    Customer->>App: Yêu cầu vào hàng mua (event_id, user_id)
    App->>Customer: Trả về captcha/challenge
    Customer->>App: Gửi lời giải challenge
    App->>DB: INSERT CHALLENGE_TOKEN(status=verified)
    App-->>Customer: Cấp quyền gửi request mua (không đảm bảo còn hàng)

    Customer->>App: Gửi request mua (event_id, user_id, idempotency_key)
    Note over App,DB: Lớp 2 - độc lập với lớp challenge ở trên, luôn chạy dù request có hay không có challenge hợp lệ
    App->>DB: BEGIN TRANSACTION
    App->>DB: UPDATE EVENT SET remaining_stock = remaining_stock - 1 WHERE id=event_id AND remaining_stock > 0
    alt còn hàng, chưa mua lần nào
        App->>DB: INSERT ORDER (user_id, event_id, idempotency_key) -- vi pham UNIQUE(event_id,user_id) neu da mua roi
        DB-->>App: COMMIT thành công
        App-->>Customer: "Đặt hàng thành công"
    else user_id đã có ORDER cho event_id này (vi phạm UNIQUE)
        DB-->>App: Lỗi vi phạm ràng buộc unique
        App->>DB: ROLLBACK
        App-->>Customer: "Bạn đã mua 1 cặp trong sự kiện này rồi"
    end
```
