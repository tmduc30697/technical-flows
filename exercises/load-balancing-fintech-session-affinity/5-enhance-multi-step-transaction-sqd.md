# Enhance sequence — Multi-step transaction (consistent hashing)

Đây là **enhance** của flow đã có ở base "Multi-step transaction (OTP)". So với base, LB không còn round-robin từng bước độc lập, mà hash `transaction_id` lên `HASH_RING` để chọn đúng một instance cố định cho toàn bộ vòng đời giao dịch, đáp ứng **yêu cầu 1** (implement consistent hashing ring, hash key là transaction_id, đảm bảo cùng key luôn map tới cùng instance trong suốt vòng đời giao dịch).

```mermaid
sequenceDiagram
    actor Customer
    participant LB as Load Balancer
    participant Ring as HASH_RING
    participant InstanceA as Instance A

    Customer->>LB: Bước 1, khởi tạo giao dịch (transaction_id=T1)
    LB->>Ring: Hash transaction_id=T1, tìm điểm ảo gần nhất trên ring
    Ring-->>LB: Trả về InstanceA (assigned_instance_id)
    LB->>LB: Ghi TRANSACTION(hash_value, assigned_instance_id=InstanceA)
    LB->>InstanceA: Route bước 1
    InstanceA->>InstanceA: Sinh OTP, lưu state
    InstanceA-->>Customer: Yêu cầu nhập OTP

    Customer->>LB: Bước 2, gửi OTP xác nhận (cùng transaction_id=T1)
    LB->>Ring: Hash lại transaction_id=T1
    Ring-->>LB: Vẫn trả về đúng InstanceA (cùng key luôn map tới cùng instance)
    LB->>InstanceA: Route bước 2 tới đúng InstanceA
    InstanceA->>InstanceA: Đọc OTP đã sinh ở bước 1 trực tiếp từ bộ nhớ, không cần đọc lại từ DB
    InstanceA-->>Customer: Xác nhận giao dịch thành công
```
