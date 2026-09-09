# Enhance sequence — Hot conversation isolation

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base vì base không có khái niệm hot conversation. Đáp ứng yêu cầu 2 của đề bài: phát hiện group chat cực lớn vượt ngưỡng và tách nó ra shard riêng để không làm quá tải shard đang chứa các conversation khác.

```mermaid
sequenceDiagram
    participant Monitor as Hot Conversation Monitor
    participant Shard2 as SHARD 2 (đang chứa nhiều conversation)
    participant DedicatedShard as SHARD dành riêng (mới)
    participant Ring as HASH_RING

    loop Định kỳ theo dõi kích thước từng conversation
        Monitor->>Shard2: Đếm số tin nhắn/ngày và tổng dung lượng theo từng conversation_id
        Shard2-->>Monitor: Conversation C_big có hàng trăm nghìn tin nhắn/ngày, vượt ngưỡng cấu hình
    end

    Monitor->>DedicatedShard: Cấp 1 shard riêng cho C_big (is_hot=true)
    Monitor->>Shard2: Đọc toàn bộ MESSAGE của C_big
    Shard2-->>Monitor: Trả về dữ liệu để di chuyển
    Monitor->>DedicatedShard: Ghi lại toàn bộ MESSAGE của C_big vào shard riêng
    Monitor->>Ring: Cập nhật ánh xạ conversation_id=C_big trỏ thẳng tới DEDICATED_SHARD (override hash ring mặc định)
    Monitor->>Shard2: Xóa dữ liệu C_big sau khi xác nhận đã copy đầy đủ
    Note over DedicatedShard: Từ giờ toàn bộ tin nhắn mới của C_big ghi thẳng vào shard riêng, không còn cạnh tranh tài nguyên với các conversation khác trên SHARD 2
```
