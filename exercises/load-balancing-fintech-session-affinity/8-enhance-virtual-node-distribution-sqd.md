# Enhance sequence — Virtual node distribution

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base không có ring nên không có khái niệm virtual node. Đáp ứng **yêu cầu 4** (cơ chế virtual node, nhiều điểm hash cho một instance, tránh phân bố lệch tải khi cluster nhỏ, ví dụ chỉ 3-4 instance).

```mermaid
sequenceDiagram
    actor Ops as Kỹ sư vận hành
    participant Ring as HASH_RING
    participant InstanceA as Instance A
    participant InstanceB as Instance B
    participant InstanceC as Instance C

    Ops->>Ring: Khởi tạo cluster chỉ với 3 instance (A, B, C)
    Ring->>Ring: Nếu mỗi instance chỉ có 1 điểm hash duy nhất, phân bố dễ lệch (ví dụ A nhận 60% traffic, B 30%, C 10%)
    Ring->>Ring: Thay vào đó, sinh nhiều virtual node cho mỗi instance (ví dụ 150 điểm hash mỗi instance)
    Ring->>InstanceA: Gán 150 VIRTUAL_NODE rải rác quanh ring
    Ring->>InstanceB: Gán 150 VIRTUAL_NODE rải rác quanh ring
    Ring->>InstanceC: Gán 150 VIRTUAL_NODE rải rác quanh ring

    loop Với một lượng lớn transaction_id ngẫu nhiên
        Ring->>Ring: Hash transaction_id, tìm virtual node gần nhất trên ring
    end
    Ring-->>Ops: Tổng hợp lại, mỗi instance nhận xấp xỉ 33% traffic, không còn lệch nhiều như khi chỉ có 1 điểm hash mỗi instance
    Note over Ring: Càng nhiều virtual node trên mỗi instance, phân bố tải càng đều, kể cả khi cluster chỉ có 3-4 instance thật
```
