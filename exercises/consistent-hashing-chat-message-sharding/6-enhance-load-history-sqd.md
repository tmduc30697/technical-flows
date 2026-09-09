# Enhance sequence — Load history (query đúng 1 shard, không scatter-gather)

Đây là **enhance**, cùng flow "load-history" đã có ở base nhưng nay thay đổi: vì toàn bộ tin nhắn của 1 conversation nằm trên đúng 1 shard, app chỉ cần tra hash ring 1 lần rồi query duy nhất shard đó, không còn scatter-gather. Đáp ứng tiếp yêu cầu 1 của đề bài (giữ tính cục bộ để đọc lịch sử nhanh).

```mermaid
sequenceDiagram
    actor User
    participant App as Chat App Server
    participant Ring as HASH_RING
    participant Shard2 as SHARD 2

    User->>App: Mở lịch sử chat conversation C1
    App->>Ring: Tra shard cho hash(conversation_id=C1)
    Ring-->>App: SHARD 2
    App->>Shard2: Query MESSAGE where conversation_id=C1
    Shard2-->>App: Trả về toàn bộ lịch sử tin nhắn của C1
    App-->>User: Hiển thị lịch sử chat
    Note over App,Shard2: Không cần join hay gộp dữ liệu từ shard khác, độ trễ không phụ thuộc vào tổng số shard trong cụm
```
