# Base sequence — Process transfer order (lock theo kho nguồn trước, thứ tự nhập trên phiếu, dẫn tới deadlock)

Đây là **base**, flow xử lý phiếu chuyển hàng ở trạng thái hiện tại: mỗi transaction lock các dòng tồn kho theo thứ tự "kho nguồn trước, rồi từng SKU theo thứ tự nhập trên phiếu", không chuẩn hoá theo composite key toàn cục. Đây chính là kịch bản deadlock kinh điển nêu ở yêu cầu 1 của đề bài: phiếu P1 (kho A sang kho B, SKU 101 rồi 102) và phiếu P2 (kho B sang kho A, SKU 102 rồi 101) chạy song song, để đơn giản minh hoạ, sơ đồ chỉ nhấn mạnh 2 lock cuối cùng hình thành vòng chờ chéo, các lock không tranh chấp khác (dòng đầu của mỗi phiếu) được lược bớt.

```mermaid
sequenceDiagram
    actor NV1 as Nhân viên (P1)
    actor NV2 as Nhân viên (P2)
    participant DB as Database
    participant InvA101 as INVENTORY (A,101)
    participant InvB101 as INVENTORY (B,101)
    participant InvA102 as INVENTORY (A,102)
    participant InvB102 as INVENTORY (B,102)

    NV1->>DB: BEGIN P1, chuyển A→B, dòng 1 = SKU101, dòng 2 = SKU102
    NV2->>DB: BEGIN P2, chuyển B→A, dòng 1 = SKU102, dòng 2 = SKU101

    DB->>InvA101: P1 dòng 1: SELECT ... FOR UPDATE (A,101) nguồn
    InvA101-->>DB: Lock granted cho P1
    DB->>InvB101: P1 dòng 1: SELECT ... FOR UPDATE (B,101) đích
    InvB101-->>DB: Lock granted cho P1, không tranh chấp

    DB->>InvB102: P2 dòng 1: SELECT ... FOR UPDATE (B,102) nguồn
    InvB102-->>DB: Lock granted cho P2
    DB->>InvA102: P2 dòng 1: SELECT ... FOR UPDATE (A,102) đích - đang được xử lý song song với P1

    DB->>InvA102: P1 dòng 2: SELECT ... FOR UPDATE (A,102) nguồn - chờ lock
    Note over InvA102: (A,102) đang bị P2 giữ hoặc đang chờ tranh chấp cùng P1

    DB->>InvB102: P1 dòng 2: SELECT ... FOR UPDATE (B,102) đích - chờ lock
    Note over InvB102: (B,102) đang bị P2 giữ lock từ dòng 1

    DB->>InvA101: P2 dòng 2: SELECT ... FOR UPDATE (A,101) đích - chờ lock
    Note over InvA101: (A,101) đang bị P1 giữ lock từ dòng 1

    Note over DB,InvB102: P1 giữ (A,101) chờ (B,102), P2 giữ (B,102) chờ (A,101), vòng chờ chéo DEADLOCK
    DB-->>NV1: Lỗi deadlock, transaction P1 bị rollback, không retry
    DB-->>NV2: Transaction P2 tiếp tục và commit thành công
    DB-->>NV1: Trả lỗi "phiếu chuyển thất bại" thẳng cho nhân viên, không log chi tiết
```
