# Enhance ERD — Khóa theo khu vực có giới hạn phạm vi/thời gian, version check, audit trail

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, thêm các trường/entity sau, ứng trực tiếp với từng yêu cầu:

- `WAREHOUSE_AREA.lock_status`, `locked_by`, `lock_started_at`, `lock_expires_at` — đáp ứng yêu cầu 2 (khóa cứng giới hạn đúng phạm vi khu vực/kệ đang kiểm, có thời gian tối đa giữ khóa, ví dụ 30 phút, kèm cảnh báo khi vượt).
- `STOCKTAKE.strategy` (pessimistic|optimistic), `snapshot_started_at` — đáp ứng yêu cầu 1 (ghi rõ chọn khóa cứng hay ghi nhận mốc thời gian để đối chiếu log sau).
- `INVENTORY_ITEM.version` — đáp ứng yêu cầu 4 (version check khi submit điều chỉnh, phát hiện số liệu đã đổi do giao dịch mới).
- `STOCKTAKE_ITEM` thêm `system_qty_at_submit`, `transactions_during_count` (danh sách `STOCK_TRANSACTION` xảy ra trong lúc đếm), `confirmed_by_staff` — đáp ứng yêu cầu 5 (audit trail đầy đủ số liệu trước/đếm/sau và các giao dịch đã tính vào chênh lệch).
- Khóa (`lock_status`/`locked_by`) được đặt ở cấp `WAREHOUSE_AREA` (vị trí vật lý), không đặt ở cấp `SKU` — đáp ứng yêu cầu 3 (ranh giới khóa theo khu vực, không theo SKU tổng, để 2 khu vực khác nhau đếm cùng SKU không chặn nhau).

```mermaid
erDiagram
    WAREHOUSE_AREA ||--o{ INVENTORY_ITEM : chứa
    SKU ||--o{ INVENTORY_ITEM : "được lưu ở"
    WAREHOUSE_AREA ||--o{ STOCK_TRANSACTION : "diễn ra tại"
    SKU ||--o{ STOCK_TRANSACTION : "liên quan"
    WAREHOUSE_AREA ||--o{ STOCKTAKE : "được kiểm ở"
    STOCKTAKE ||--o{ STOCKTAKE_ITEM : "gồm các dòng đếm"
    SKU ||--o{ STOCKTAKE_ITEM : "được đếm"
    STOCKTAKE_ITEM ||--o{ STOCK_TRANSACTION : "tính vào chênh lệch"

    WAREHOUSE_AREA {
        string id PK
        string name
        string lock_status "free|locked"
        string locked_by "staff_id đang giữ khóa, null nếu free"
        datetime lock_started_at
        datetime lock_expires_at "vd: lock_started_at + 30 phút"
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
        int version "optimistic lock cho submit điều chỉnh"
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
        string strategy "pessimistic_lock|optimistic_reconcile"
        datetime started_at
        datetime snapshot_started_at "mốc để đối chiếu log giao dịch"
        string status "in_progress|awaiting_confirmation|submitted"
    }
    STOCKTAKE_ITEM {
        string id PK
        string stocktake_id FK
        string sku_id FK
        int counted_qty
        int system_qty_before "số hệ thống lúc bắt đầu đếm"
        int system_qty_at_submit "số hệ thống ngay lúc submit, dùng version check"
        int adjusted_qty "số cuối cùng sau điều chỉnh"
        string transactions_during_count "danh sách STOCK_TRANSACTION.id xảy ra trong lúc đếm"
        boolean confirmed_by_staff "nhân viên đã xác nhận lại phần chênh lệch"
    }
```
