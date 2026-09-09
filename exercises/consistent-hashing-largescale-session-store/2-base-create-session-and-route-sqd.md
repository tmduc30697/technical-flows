# Base sequence — Create session and route (1 điểm ring mỗi node, không replicate)

Đây là **base**, flow tạo session và xác định node lưu trữ. Ring chỉ có đúng 1 điểm hash cho mỗi node vật lý (không có virtual node), nên phân bổ tải có thể lệch giữa các node, và session chỉ được ghi trên đúng 1 node duy nhất — không có bản sao nào khác. Đây là nền cho yêu cầu 1 (cần virtual node để tránh lệch tải) và yêu cầu 3 (cần replication để chịu được mất node) của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant App as Login Service
    participant Ring as Hash Ring (1 điểm/node)
    participant NodeA as NODE A

    User->>App: Đăng nhập thành công
    App->>Ring: Tra node cho hash(user_id)
    Ring-->>App: Rơi vào NODE A
    App->>NodeA: Ghi SESSION(user_id, ttl_seconds=3600)
    NodeA-->>App: Lưu thành công
    App-->>User: Trả session token
    Note over Ring,NodeA: Vì mỗi node chỉ có 1 điểm trên ring, dải hash gán cho NODE A có thể lớn hơn hẳn các node khác, dồn nhiều session hơn vào đúng NODE A
    Note over NodeA: Session này chỉ tồn tại duy nhất trên NODE A, không có bản sao ở node nào khác
```
