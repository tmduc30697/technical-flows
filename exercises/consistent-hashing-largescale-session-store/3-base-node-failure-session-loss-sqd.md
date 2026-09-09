# Base sequence — Node failure (mất session hàng loạt)

Đây là **base**, flow khi 1 node lưu session chết đột ngột. Vì consistent hashing thuần không có replication, toàn bộ session trên node chết dồn hết sang node kế tiếp trên ring theo định nghĩa thuật toán — nhưng dữ liệu thực tế của các session đó đã mất, không tự nhiên "có sẵn" ở node kế tiếp. Đây là vấn đề nền cho yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant Ring as Hash Ring
    participant NodeA as NODE A (chết)
    participant NodeB as NODE B (kế tiếp trên ring)

    Note over NodeA: NODE A chết đột ngột, đang giữ hàng loạt session của 1 dải hash
    Ring->>Ring: Phát hiện NODE A không còn healthy, loại khỏi ring
    Ring->>Ring: Dải hash trước đó thuộc NODE A nay tự động thuộc về NODE B (node kế tiếp)

    User->>NodeB: Request kèm session token (session vốn tạo trên NODE A)
    NodeB->>NodeB: Tra SESSION theo user_id, không tìm thấy (dữ liệu chưa từng tồn tại ở đây)
    NodeB-->>User: session not found, yêu cầu đăng nhập lại

    Note over NodeA,NodeB: Toàn bộ user rơi vào đúng dải hash của NODE A bị đăng xuất hàng loạt cùng lúc, dù bản thân họ không làm gì sai
```
