# Base sequence — Send message (route theo message_id, không theo conversation)

Đây là **base**, flow gửi tin nhắn ở trạng thái hiện tại — shard được chọn bằng cách hash/round-robin theo `message_id` (mỗi tin nhắn mới có thể rơi vào 1 shard bất kỳ). Đây là nguyên nhân khiến tin nhắn của cùng 1 conversation bị rải trên nhiều shard, làm nền cho yêu cầu 1 của đề bài (phải luôn về đúng 1 shard theo conversation_id).

```mermaid
sequenceDiagram
    actor User
    participant App as Chat App Server
    participant Router as Message Router (hash theo message_id)
    participant Shard1 as SHARD 1
    participant Shard2 as SHARD 2

    User->>App: Gửi tin nhắn vào conversation C1
    App->>App: Sinh message_id mới
    App->>Router: Chọn shard cho message_id vừa sinh
    Router->>Router: hash(message_id) mod N
    Router-->>App: Kết quả trỏ tới SHARD 2
    App->>Shard2: Ghi MESSAGE(conversation_id=C1, shard_id=2)
    Note over Shard1,Shard2: Tin nhắn trước đó của cùng conversation C1 (message_id khác) có thể đã ghi vào SHARD 1 do hash message_id khác nhau
    Shard2-->>User: Gửi thành công
```
