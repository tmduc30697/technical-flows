# Enhance sequence — Client retry do mạng lag, chống xử lý trùng bằng idempotency key

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 3** của đề bài: mô tả cụ thể trường hợp 1 user đã vượt challenge, gửi đúng 1 request mua, nhưng do mạng lag client tự động retry gửi thêm 2 request giống hệt trong vòng 1 giây. Hệ thống phải nhận diện cả 3 request mang cùng `idempotency_key` và chỉ xử lý 1 lần duy nhất.

```mermaid
sequenceDiagram
    actor Client as Client (browser/app)
    participant App as Checkout Service
    participant DB as ORDER store

    Client->>App: POST /purchase (idempotency_key=abc123)
    Note over Client: Không thấy response kịp trong 300ms do mạng lag
    Client->>App: Retry POST /purchase (idempotency_key=abc123)
    Client->>App: Retry lần 2 POST /purchase (idempotency_key=abc123)

    par Request gốc xử lý trước
        App->>DB: SELECT ORDER WHERE idempotency_key=abc123
        DB-->>App: Không tìm thấy
        App->>DB: INSERT ORDER (idempotency_key=abc123, status=pending), trừ tồn kho trong cùng transaction
        DB-->>App: COMMIT thành công
        App-->>Client: 201 Created (order_id=xyz)
    and Request retry 1 tới gần như đồng thời
        App->>DB: SELECT ORDER WHERE idempotency_key=abc123 FOR UPDATE
        DB-->>App: Đã tồn tại (do transaction gốc đã commit hoặc đang giữ lock)
        App-->>Client: 200 OK (trả lại order_id=xyz đã tạo, không tạo mới, không trừ tồn kho thêm)
    and Request retry 2 tới sau
        App->>DB: SELECT ORDER WHERE idempotency_key=abc123
        DB-->>App: Đã tồn tại
        App-->>Client: 200 OK (trả lại order_id=xyz)
    end
    Note over App,DB: Cả 3 request chỉ trừ tồn kho đúng 1 lần và chỉ tạo đúng 1 ORDER, thẻ chỉ bị charge 1 lần
```
