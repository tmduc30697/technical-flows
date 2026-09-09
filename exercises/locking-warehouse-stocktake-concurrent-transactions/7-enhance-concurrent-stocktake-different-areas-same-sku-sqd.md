# Enhance sequence — 2 nhân viên kiểm kho khác khu vực nhưng cùng SKU tổng

Đây là **enhance**, flow mới minh họa ranh giới khóa của hệ thống sau khi áp dụng khóa cứng theo khu vực. Hai nhân viên kiểm kho ở khu vực A và khu vực B khác nhau, cùng đếm SKU X (cùng SKU tổng lưu ở 2 kệ khác nhau) — vì khóa đặt ở cấp `WAREHOUSE_AREA` chứ không đặt ở cấp `SKU`, 2 nhân viên không hề chặn lẫn nhau. Đáp ứng yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor NV1 as Nhân viên 1
    actor NV2 as Nhân viên 2
    participant App as Stocktake App
    participant DB as Database

    NV1->>App: Bắt đầu kiểm kho khu vực A (đếm SKU X)
    App->>DB: UPDATE WAREHOUSE_AREA SET lock_status=locked, locked_by=NV1 WHERE id=A
    DB-->>App: OK, chỉ area A bị khóa

    NV2->>App: Bắt đầu kiểm kho khu vực B (cũng đếm SKU X, cùng SKU tổng nhưng khác kệ)
    App->>DB: UPDATE WAREHOUSE_AREA SET lock_status=locked, locked_by=NV2 WHERE id=B
    DB-->>App: OK, area B độc lập với area A, không bị chặn bởi khóa của NV1

    Note over App,DB: Nếu khóa được đặt theo SKU tổng thay vì theo khu vực, NV2 sẽ bị chặn dù đang đếm ở kệ hoàn toàn khác, gây chờ đợi không cần thiết

    NV1->>App: Submit số đếm khu vực A, SKU X = 30
    App->>DB: UPDATE INVENTORY_ITEM SET quantity=30 WHERE area=A, sku=X
    App->>DB: UPDATE WAREHOUSE_AREA SET lock_status=free WHERE id=A

    NV2->>App: Submit số đếm khu vực B, SKU X = 20 (độc lập, không phụ thuộc vào NV1)
    App->>DB: UPDATE INVENTORY_ITEM SET quantity=20 WHERE area=B, sku=X
    App->>DB: UPDATE WAREHOUSE_AREA SET lock_status=free WHERE id=B

    Note over DB: Tổng tồn kho SKU X trên toàn hệ thống = 30 (area A) + 20 (area B) = 50, cả 2 kiểm kho hoàn tất song song không hề chờ nhau
```
