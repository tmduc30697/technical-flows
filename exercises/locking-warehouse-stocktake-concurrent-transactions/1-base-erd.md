# Base ERD — Quản lý kho có xuất/nhập, chưa có kiểm soát tương tranh cho kiểm kho

Đây là **base**: trạng thái hệ thống quản lý kho *trước khi* áp dụng cơ chế khóa/version cho flow kiểm kho. Suy luận từ đề bài, base đã có `WAREHOUSE_AREA` (khu vực/kệ hàng), `SKU` (mã hàng), `INVENTORY_ITEM` lưu số lượng tồn của 1 SKU tại 1 khu vực cụ thể, và `STOCK_TRANSACTION` ghi lại mọi phiếu xuất/nhập kho làm thay đổi `INVENTORY_ITEM.quantity`. Đã có tính năng `STOCKTAKE` để nhân viên ghi nhận số đếm thực tế, nhưng chưa có bất kỳ khóa hay version check nào — số đếm được ghi đè thẳng vào hệ thống, không đối chiếu với các giao dịch xảy ra trong lúc đếm.

```mermaid
erDiagram
    WAREHOUSE_AREA ||--o{ INVENTORY_ITEM : chứa
    SKU ||--o{ INVENTORY_ITEM : "được lưu ở"
    WAREHOUSE_AREA ||--o{ STOCK_TRANSACTION : "diễn ra tại"
    SKU ||--o{ STOCK_TRANSACTION : "liên quan"
    WAREHOUSE_AREA ||--o{ STOCKTAKE : "được kiểm ở"
    STOCKTAKE ||--o{ STOCKTAKE_ITEM : "gồm các dòng đếm"
    SKU ||--o{ STOCKTAKE_ITEM : "được đếm"

    WAREHOUSE_AREA {
        string id PK
        string name
    }
    SKU {
        string id PK
        string name
    }
    INVENTORY_ITEM {
        string id PK
        string area_id FK
        string sku_id FK
        int quantity
    }
    STOCK_TRANSACTION {
        string id PK
        string area_id FK
        string sku_id FK
        string type "inbound|outbound"
        int qty_delta
        datetime created_at
    }
    STOCKTAKE {
        string id PK
        string area_id FK
        string staff_id
        datetime started_at
        string status "in_progress|submitted"
    }
    STOCKTAKE_ITEM {
        string id PK
        string stocktake_id FK
        string sku_id FK
        int counted_qty
        int system_qty_before "số hệ thống ghi nhận, đọc tại thời điểm submit"
        int adjusted_qty
    }
```
