# Enhance sequence — Send message (route theo consistent hash của conversation_id)

Đây là **enhance**, cùng flow "send-message" đã có ở base nhưng nay thay đổi: shard được xác định bằng consistent hash của `conversation_id` (không phải message_id), nên mọi tin nhắn của cùng 1 conversation luôn về đúng 1 shard cố định. Đáp ứng trực tiếp yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant App as Chat App Server
    participant Ring as HASH_RING
    participant Shard2 as SHARD 2

    User->>App: Gửi tin nhắn vào conversation C1
    App->>Ring: Tra shard cho hash(conversation_id=C1)
    Ring->>Ring: Tìm virtual node gần nhất theo chiều kim đồng hồ trên ring
    Ring-->>App: conversation_id=C1 luôn thuộc SHARD 2 (kết quả xác định, ổn định)
    App->>Shard2: Ghi MESSAGE(conversation_id=C1)
    Shard2-->>User: Gửi thành công
    Note over Ring,Shard2: Mọi tin nhắn tiếp theo của C1, dù gửi từ session/app instance nào, đều tra ra cùng SHARD 2 vì cùng dùng chung 1 hash ring
```
