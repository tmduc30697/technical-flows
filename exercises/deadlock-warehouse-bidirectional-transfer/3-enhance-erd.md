# Enhance ERD — Thêm thứ tự lock toàn cục, retry đọc lại tồn kho, isolation level tường minh và deadlock logging

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, `TRANSFER_ORDER_LINE` có thêm trường `lock_sequence` (thứ tự lock đã chuẩn hoá theo composite key `(warehouse_id, sku_id)` tăng dần, tách biệt với `line_no` là thứ tự nhập gốc trên phiếu), `TRANSFER_ORDER` thêm `isolation_level`, `retry_count`, `max_retry`, và có thêm entity mới `DEADLOCK_LOG`, ứng trực tiếp với các yêu cầu:

- `TRANSFER_ORDER_LINE.lock_sequence` — đáp ứng yêu cầu 1 (chuẩn hoá thứ tự lock toàn cục theo composite key, bất kể chiều chuyển hay thứ tự nhập) và yêu cầu 2 (lock hết toàn bộ dòng nguồn lẫn đích theo thứ tự này trước khi xử lý bất kỳ dòng nào).
- `TRANSFER_ORDER.isolation_level` — đáp ứng yêu cầu 4 (chọn `READ COMMITTED` tường minh thay vì mặc định `REPEATABLE READ`).
- `TRANSFER_ORDER.retry_count`/`max_retry` — đáp ứng yêu cầu 3 (retry tự động, đọc lại tồn kho hiện tại tại thời điểm retry).
- `DEADLOCK_LOG` (mới) — ghi chi tiết mỗi lần deadlock xảy ra dù đã chuẩn hoá thứ tự lock, dữ liệu này cũng phục vụ test invariant ở yêu cầu 5.

```mermaid
erDiagram
    WAREHOUSE ||--o{ INVENTORY : "có tồn kho tại"
    SKU ||--o{ INVENTORY : "được lưu tồn"
    WAREHOUSE ||--o{ TRANSFER_ORDER : "là kho nguồn"
    WAREHOUSE ||--o{ TRANSFER_ORDER : "là kho đích"
    TRANSFER_ORDER ||--o{ TRANSFER_ORDER_LINE : "gồm nhiều dòng SKU"
    SKU ||--o{ TRANSFER_ORDER_LINE : "được chuyển"
    TRANSFER_ORDER ||--o{ DEADLOCK_LOG : "có thể phát sinh"

    WAREHOUSE {
        string id PK
        string name
    }
    SKU {
        string id PK
        string name
    }
    INVENTORY {
        string warehouse_id FK
        string sku_id FK
        int quantity
    }
    TRANSFER_ORDER {
        string id PK
        string from_warehouse_id FK
        string to_warehouse_id FK
        string status "pending|success|failed"
        string isolation_level "READ_COMMITTED"
        int retry_count
        int max_retry "3"
        datetime created_at
    }
    TRANSFER_ORDER_LINE {
        string id PK
        string transfer_order_id FK
        string sku_id FK
        int quantity
        int line_no "thứ tự nhập gốc trên phiếu"
        int lock_sequence "thứ tự lock đã chuẩn hoá theo (warehouse_id, sku_id) tăng dần"
    }
    DEADLOCK_LOG {
        string id PK
        string transfer_order_id FK
        string blocking_transfer_order_id
        string locked_resource "vd: warehouse_id=B,sku_id=102"
        string db_error_code "40P01 | 1213"
        int retry_attempt
        datetime occurred_at
    }
```
