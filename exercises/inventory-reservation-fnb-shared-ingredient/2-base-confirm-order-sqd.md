# Base sequence — Xác nhận đơn và trừ tồn nguyên liệu

Đây là **base**, mô tả flow xác nhận đơn hàng ở trạng thái hiện tại: hệ thống trừ tồn nguyên liệu tuần tự theo từng món trong đơn, không có khoá/điều kiện atomic, không kiểm tra đủ số lượng trước khi trừ theo kiểu nguyên tử. Đây chính là flow sẽ bị race condition khi 2 đơn từ 2 kênh khác nhau cùng cần chung 1 nguyên liệu gần hết, và là tiền đề để enhance thêm cơ chế atomic conditional update (yêu cầu 1 và 2 trong đề bài).

```mermaid
sequenceDiagram
    actor App as Khach dat qua App
    actor Instore as Khach tai cho
    participant OrderSvc as Order Service
    participant Stock as Ingredient Stock (DB)
    participant Kitchen as Bep (phieu che bien)

    App->>OrderSvc: Dat mon "Pho bo" (don 1)
    Instore->>OrderSvc: Dat mon "Bun bo" (don 2)

    OrderSvc->>Stock: Doc ton "thit bo" hien tai
    Stock-->>OrderSvc: stock_quantity = du cho 1 suat

    OrderSvc->>Stock: Doc lai ton "thit bo" (cho don 2)
    Stock-->>OrderSvc: stock_quantity = du cho 1 suat (chua cap nhat kip)

    par Xu ly tuan tu tung dong don, khong khoa
        OrderSvc->>Stock: UPDATE stock_quantity = stock_quantity - so_luong (don 1)
        Stock-->>OrderSvc: OK, tru thanh cong
    and
        OrderSvc->>Stock: UPDATE stock_quantity = stock_quantity - so_luong (don 2)
        Stock-->>OrderSvc: OK, tru thanh cong (nhung thuc te am ton hoac vuot qua ton that)
    end

    OrderSvc->>Kitchen: In phieu che bien cho ca 2 don
    Note over OrderSvc,Kitchen: Ca 2 don deu duoc xac nhan du kho khong du,<br/>phat hien thieu nguyen lieu chi khi bep bao lai thu cong
```
