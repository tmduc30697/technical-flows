# Enhance sequence — Xác nhận đơn với atomic reservation nhiều nguyên liệu

Đây là **enhance** của flow `confirm-order` đã có ở base. So với base (trừ tồn tuần tự từng dòng, không khoá), flow này thay đổi ở chỗ: toàn bộ nguyên liệu của toàn bộ món trong 1 đơn được trừ trong 1 transaction atomic duy nhất bằng conditional update (`WHERE stock_quantity >= quantity_required AND version = current_version`), đáp ứng đúng kịch bản đề bài — đơn 1 (app, "Phở bò") và đơn 2 (tại chỗ, "Bún bò") cùng cần "thịt bò" nhưng kho chỉ đủ 1 suất (yêu cầu 1), và nếu 1 nguyên liệu bất kỳ trong đơn không đủ thì toàn bộ đơn rollback, không rơi vào trạng thái nửa xác nhận (yêu cầu 2).

```mermaid
sequenceDiagram
    actor App as Khach dat qua App (don 1, Pho bo)
    actor Instore as Khach tai cho (don 2, Bun bo)
    participant OrderSvc as Order Service
    participant Stock as Ingredient Stock (DB, transactional)
    participant Kitchen as Bep

    par 2 don den gan nhu dong thoi
        App->>OrderSvc: Xac nhan don 1 (Pho bo can 200g thit bo)
    and
        Instore->>OrderSvc: Xac nhan don 2 (Bun bo can 200g thit bo)
    end

    OrderSvc->>Stock: BEGIN TRANSACTION (don 1), tao RESERVATION cho toan bo mon trong don
    OrderSvc->>Stock: UPDATE thit_bo SET stock -= 200, version += 1 WHERE stock >= 200 AND version = v1
    Stock-->>OrderSvc: 1 row affected, thanh cong
    OrderSvc->>Stock: COMMIT (don 1), RESERVATION.status = reserved
    OrderSvc-->>App: Don duoc xac nhan

    OrderSvc->>Stock: BEGIN TRANSACTION (don 2)
    OrderSvc->>Stock: UPDATE thit_bo SET stock -= 200, version += 1 WHERE stock >= 200 AND version = v2
    Stock-->>OrderSvc: 0 row affected (stock hien tai khong con du 200g)
    OrderSvc->>Stock: ROLLBACK (don 2), khong tao RESERVATION nao
    OrderSvc-->>Instore: Tu choi ngay, "Bun bo het nguyen lieu, mon tam ngung phuc vu"

    Note over OrderSvc,Kitchen: Chi don da co RESERVATION.status = reserved moi duoc gui phieu xuong bep
    OrderSvc->>Kitchen: In phieu che bien cho don 1

    Note over OrderSvc,Stock: Neu don gom nhieu mon dung chung nhieu nguyen lieu, tat ca cac dong UPDATE deu nam trong cung 1 transaction, 1 dong that bai la toan bo rollback, khong co mon nao bi xac nhan rieng le
```
