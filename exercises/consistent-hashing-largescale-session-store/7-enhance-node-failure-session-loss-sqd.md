# Enhance sequence — Node failure (replica đã có sẵn, không mất session)

Đây là **enhance**, cùng flow "node-failure-session-loss" đã có ở base nhưng nay thay đổi hoàn toàn: vì mỗi session đã được replicate sang N node kế cận từ trước (xem flow "create-session-and-route"), node chết không còn gây mất session — 1 trong các replica còn sống được promote lên làm primary ngay. Đáp ứng yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant Ring as Hash Ring
    participant NodeA as NODE A (chết, vốn là primary)
    participant NodeB as NODE B (đã có replica từ trước)

    Note over NodeA: NODE A chết đột ngột, đang giữ vai trò primary cho hàng loạt session
    Ring->>Ring: Phát hiện NODE A không healthy, loại khỏi ring
    Ring->>NodeB: Với mỗi session vốn có replica trên NodeB, promote SESSION_REPLICA(role=replica) thành role=primary

    User->>NodeB: Request kèm session token (session vốn có primary trên NODE A)
    NodeB->>NodeB: Tra SESSION_REPLICA theo user_id, tìm thấy bản đã được promote, expires_at giữ nguyên như bản gốc
    NodeB-->>User: Phục vụ request bình thường, không đăng xuất

    Ring->>Ring: Khi có node thay thế NODE A, tự động replicate lại để đủ N bản sao trở lại
    Note over NodeA,NodeB: Chỉ những session mà toàn bộ N replica cùng nằm trên các node chết đồng thời mới thực sự bị mất, đây là số liệu cần đo lường để đánh giá hiệu quả chiến lược replication
```
