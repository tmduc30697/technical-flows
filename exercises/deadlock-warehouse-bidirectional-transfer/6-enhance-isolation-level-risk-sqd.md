# Enhance sequence — Vì sao chọn READ COMMITTED thay vì REPEATABLE READ mặc định

Đây là **enhance**, sơ đồ minh hoạ lý do lựa chọn kỹ thuật chứ không phải 1 flow nghiệp vụ đơn thuần, đáp ứng yêu cầu 4 của đề bài: giải thích cụ thể vì sao `REPEATABLE READ` (mặc định của một số DB như MySQL/InnoDB) có thể mở rộng phạm vi lock qua gap lock/range lock, ảnh hưởng tới các phiếu chuyển kho khác không hề liên quan tới cùng SKU, làm tăng deadlock giả tạo — trong khi `READ COMMITTED` chỉ lock đúng các dòng đã match.

```mermaid
sequenceDiagram
    actor NV1 as Nhân viên (P1: A→B, SKU101+102)
    actor NV3 as Nhân viên (P4: C→D, SKU999, không liên quan gì tới P1)
    participant DB as Database
    participant Inv as Bảng INVENTORY

    rect rgb(255, 230, 230)
    Note over DB,Inv: Kịch bản nếu dùng REPEATABLE READ mặc định (không mong muốn)
    NV1->>DB: P1: SELECT ... FOR UPDATE WHERE warehouse_id IN (A,B) AND sku_id IN (101,102)
    DB->>Inv: Range scan theo điều kiện, đặt gap lock/next-key lock bao trùm cả khoảng giữa các dòng lân cận trong index
    Inv-->>DB: Lock granted cho P1, nhưng phạm vi lock rộng hơn 4 dòng thực sự cần
    NV3->>DB: P4: SELECT ... FOR UPDATE WHERE warehouse_id IN (C,D) AND sku_id IN (999) - dòng nằm trong gap bị P1 khoá
    DB-->>NV3: BLOCKED dù P4 không hề đụng SKU hay kho nào trùng với P1
    Note over DB,Inv: P4 chờ P1 giải phóng gap lock dù 2 phiếu hoàn toàn độc lập, tăng khả năng timeout/deadlock giả tạo với các phiếu khác đang chờ P4
    end

    rect rgb(230, 255, 230)
    Note over DB,Inv: Sau enhance, dùng READ COMMITTED tường minh
    NV1->>DB: P1: SELECT ... FOR UPDATE WHERE (warehouse_id, sku_id) IN ((A,101),(A,102),(B,101),(B,102))
    DB->>Inv: Chỉ đặt row lock đúng 4 dòng match điều kiện, không mở rộng gap lock
    Inv-->>DB: Lock granted cho P1, đúng phạm vi cần
    NV3->>DB: P4: SELECT ... FOR UPDATE WHERE (warehouse_id, sku_id) = (C,999) / (D,999)
    DB-->>NV3: Lock granted ngay lập tức, không hề bị chặn bởi P1
    Note over DB,Inv: 2 phiếu không liên quan chạy hoàn toàn độc lập, chỉ còn deadlock thật giữa các phiếu thực sự chồng lấn SKU
    end
```
