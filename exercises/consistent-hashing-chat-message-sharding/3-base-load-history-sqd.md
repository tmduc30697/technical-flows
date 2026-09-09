# Base sequence — Load history (scatter-gather nhiều shard)

Đây là **base**, flow tải lịch sử tin nhắn của 1 conversation. Vì tin nhắn của conversation đó có thể rải trên nhiều shard (do route theo message_id ở flow "send-message"), app phải scatter-gather — truy vấn tất cả shard rồi gộp lại — tốn kém và chậm. Đây chính là vấn đề mà yêu cầu 1 của đề bài muốn loại bỏ.

```mermaid
sequenceDiagram
    actor User
    participant App as Chat App Server
    participant Shard1 as SHARD 1
    participant Shard2 as SHARD 2
    participant ShardN as SHARD N

    User->>App: Mở lịch sử chat conversation C1
    par Scatter tới toàn bộ shard vì không biết chắc tin nhắn nằm ở đâu
        App->>Shard1: Query MESSAGE where conversation_id=C1
        App->>Shard2: Query MESSAGE where conversation_id=C1
        App->>ShardN: Query MESSAGE where conversation_id=C1
    end
    Shard1-->>App: Trả về 1 phần tin nhắn
    Shard2-->>App: Trả về 1 phần tin nhắn
    ShardN-->>App: Không có tin nhắn nào của C1
    App->>App: Gather, merge, sắp xếp lại theo thời gian từ nhiều nguồn
    App-->>User: Hiển thị lịch sử chat
    Note over App: Chi phí query tăng tuyến tính theo số shard, càng nhiều shard càng chậm dù conversation chỉ có ít tin nhắn
```
