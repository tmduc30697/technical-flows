# Enhance sequence — Instance failure handling (failover + khôi phục state)

Đây là **enhance** của flow đã có ở base "Instance failure handling". So với base, state được ghi bền vững vào `TRANSACTION_STATE_SNAPSHOT` sau mỗi bước, và khi instance đang giữ giao dịch chết, LB phát hiện, remap ring, chuyển request tiếp theo sang instance khác kèm khôi phục state, đáp ứng **yêu cầu 2** (phát hiện instance chết giữa giao dịch, fail-over kèm khôi phục state từ nguồn bền, không để giao dịch treo vô thời hạn).

```mermaid
sequenceDiagram
    actor Customer
    participant LB as Load Balancer
    participant Ring as HASH_RING
    participant InstanceA as Instance A (đang giữ giao dịch)
    participant InstanceC as Instance C (thay thế)
    participant Snapshot as TRANSACTION_STATE_SNAPSHOT (bền vững)
    participant Monitor as Instance Health Monitor

    Customer->>LB: Bước 1, khởi tạo giao dịch (transaction_id=T2)
    LB->>Ring: Hash T2, chọn InstanceA
    LB->>InstanceA: Route bước 1
    InstanceA->>Snapshot: Ghi state (OTP, retry_count) bền vững ngay sau khi sinh, không chỉ giữ trong bộ nhớ
    InstanceA-->>Customer: Yêu cầu nhập OTP

    Note over InstanceA: Instance A gặp sự cố, tiến trình chết đột ngột
    Monitor->>InstanceA: Heartbeat check
    InstanceA--xMonitor: Không phản hồi
    Monitor->>Monitor: Xác nhận InstanceA down, ghi INSTANCE_FAILOVER_EVENT(detected_at=now)
    Monitor->>Ring: Đánh dấu InstanceA down, remap tạm thời các virtual node của nó

    Customer->>LB: Bước 2, gửi OTP xác nhận (transaction_id=T2)
    LB->>Ring: Hash T2, vì InstanceA down nên trả về InstanceC (instance sống gần nhất trên ring)
    LB->>InstanceC: Route bước 2 sang InstanceC
    InstanceC->>Snapshot: Đọc lại state đã ghi bền vững của T2
    Snapshot-->>InstanceC: Trả OTP và retry_count đã lưu
    InstanceC->>Monitor: Ghi state_restored_at vào INSTANCE_FAILOVER_EVENT
    InstanceC-->>Customer: Tiếp tục xử lý xác nhận OTP bình thường, giao dịch không bị treo
```
