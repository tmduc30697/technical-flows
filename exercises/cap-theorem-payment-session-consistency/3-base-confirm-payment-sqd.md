# Base sequence — Confirm payment (dùng chung eventual consistency, double-charge)

Đây là **base**, flow "Xác nhận thanh toán" ở trạng thái hiện tại — dùng cùng cơ chế eventual consistency như mọi API khác, không có khóa hay quorum riêng. Flow này liên quan mật thiết tới enhance vì toàn bộ 5 yêu cầu của đề bài đều nhằm sửa đúng lỗ hổng nguy hiểm này.

```mermaid
sequenceDiagram
    actor UserRequest1 as Request xác nhận thanh toán (lần 1)
    actor UserRequest2 as Request xác nhận thanh toán (lần 2, trùng, network chập chờn)
    participant NodeA as SESSION_REPLICA node A
    participant NodeB as SESSION_REPLICA node B
    participant Gateway as Payment Gateway

    UserRequest1->>NodeA: Đọc status session (node A, có thể chưa thấy update từ node khác)
    NodeA-->>UserRequest1: status=pending
    UserRequest2->>NodeB: Đọc status session (node B, cũng chưa thấy update)
    NodeB-->>UserRequest2: status=pending

    UserRequest1->>Gateway: Charge tiền lần 1
    Gateway-->>UserRequest1: Thành công
    UserRequest1->>NodeA: Cập nhật status=confirmed

    UserRequest2->>Gateway: Charge tiền lần 2 (vẫn tưởng đang pending)
    Gateway-->>UserRequest2: Thành công
    UserRequest2->>NodeB: Cập nhật status=confirmed

    Note over NodeA,NodeB: 2 node cuối cùng hội tụ về cùng 1 giá trị (eventual consistency), nhưng khách hàng đã bị charge 2 lần — không có bước từ chối/khóa nào ngăn được việc này
```
