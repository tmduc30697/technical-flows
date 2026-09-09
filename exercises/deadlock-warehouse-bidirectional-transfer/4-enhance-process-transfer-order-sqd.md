# Enhance sequence — Process transfer order (lock toàn bộ dòng theo composite key toàn cục, loại trừ deadlock)

Đây là **enhance** của flow `process-transfer-order` đã có ở base. So với base (lock rải rác theo "nguồn trước, thứ tự nhập trên phiếu" gây deadlock), nay mỗi transaction gom toàn bộ dòng liên quan (cả nguồn lẫn đích) thành 1 danh sách composite key `(warehouse_id, sku_id)`, sắp xếp tăng dần, rồi lock hết theo đúng thứ tự này ngay từ đầu trước khi trừ/cộng bất kỳ dòng nào. Đáp ứng yêu cầu 1 và 2 của đề bài.

```mermaid
sequenceDiagram
    actor NV1 as Nhân viên (P1)
    actor NV2 as Nhân viên (P2)
    participant DB as Database (isolation=READ COMMITTED)
    participant InvA101 as INVENTORY (A,101)
    participant InvA102 as INVENTORY (A,102)
    participant InvB101 as INVENTORY (B,101)
    participant InvB102 as INVENTORY (B,102)

    NV1->>DB: BEGIN P1, chuyển A→B, dòng SKU101+SKU102
    NV1->>DB: Gom composite key liên quan {(A,101),(A,102),(B,101),(B,102)}, sort tăng dần
    NV2->>DB: BEGIN P2, chuyển B→A, dòng SKU102+SKU101
    NV2->>DB: Gom composite key liên quan {(B,101),(B,102),(A,101),(A,102)}, sort tăng dần = cùng thứ tự với P1

    DB->>InvA101: P1: SELECT ... FOR UPDATE (A,101) theo lock_sequence
    InvA101-->>DB: Lock granted cho P1

    DB->>InvA101: P2: SELECT ... FOR UPDATE (A,101) theo lock_sequence - chờ lock
    Note over InvA101: P2 chờ P1 giải phóng (A,101), chưa hề đụng tới bất kỳ resource nào khác

    DB->>InvA102: P1: SELECT ... FOR UPDATE (A,102) theo lock_sequence
    InvA102-->>DB: Lock granted cho P1
    DB->>InvB101: P1: SELECT ... FOR UPDATE (B,101) theo lock_sequence
    InvB101-->>DB: Lock granted cho P1
    DB->>InvB102: P1: SELECT ... FOR UPDATE (B,102) theo lock_sequence
    InvB102-->>DB: Lock granted cho P1

    DB->>DB: P1 đã giữ đủ 4 lock, bắt đầu trừ tồn kho A, cộng tồn kho B, COMMIT
    DB->>InvA101: Giải phóng toàn bộ lock của P1

    DB->>InvA101: P2 nhận lock (A,101) vừa được giải phóng
    InvA101-->>DB: Lock granted cho P2
    DB->>InvA102: P2: SELECT ... FOR UPDATE (A,102)
    InvA102-->>DB: Lock granted cho P2
    DB->>InvB101: P2: SELECT ... FOR UPDATE (B,101)
    InvB101-->>DB: Lock granted cho P2
    DB->>InvB102: P2: SELECT ... FOR UPDATE (B,102)
    InvB102-->>DB: Lock granted cho P2
    DB->>DB: P2 đã giữ đủ 4 lock, bắt đầu trừ tồn kho B, cộng tồn kho A, COMMIT

    Note over DB,InvB102: Vì cả P1 và P2 đều lock theo đúng cùng 1 thứ tự composite key toàn cục, giao dịch đến sau chỉ chờ giao dịch đến trước, không bao giờ hình thành vòng chờ chéo
```
