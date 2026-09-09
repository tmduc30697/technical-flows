# Sequence - Enhance - Flow "inventory-sync"

Đây là **enhance**, cùng flow `inventory-sync` như ở base nhưng thay ghi đè trực tiếp bằng update nguyên tử có điều kiện, đáp ứng yêu cầu 3: đồng bộ tồn kho từ nhà cung cấp không được ghi đè mất các thay đổi tạm thời (`reserved_count`) do order đang xử lý song song tạo ra trong lúc đồng bộ diễn ra.

```mermaid
sequenceDiagram
    participant Scheduler as Sync Scheduler
    participant SupplierAPI as API Nhà cung cấp
    participant Cache as Inventory Cache
    participant App as Sàn dropshipping

    par Order khác đang xử lý song song
        App->>Cache: UPDATE reserved_count = reserved_count + 1, version += 1
    and Sync định kỳ diễn ra cùng lúc
        Scheduler->>SupplierAPI: lấy tồn kho mới nhất cho sản phẩm X
        SupplierAPI-->>Scheduler: quantity thật = 3
        Scheduler->>Cache: đọc version hiện tại trước khi ghi
        Cache-->>Scheduler: version = v
        Scheduler->>Cache: UPDATE cached_quantity = 3, version = v + 1 WHERE version = v
        alt Version còn khớp (không ai ghi xen giữa)
            Cache-->>Scheduler: update thành công, reserved_count của order song song vẫn nguyên vẹn
        else Version đã đổi (order song song vừa ghi trước)
            Cache-->>Scheduler: update thất bại do version lệch
            Scheduler->>Cache: đọc lại version mới, thử ghi lại cached_quantity = 3
        end
    end

    Note over Cache: reserved_count không bao giờ bị ghi đè về 0,\nchỉ cached_quantity (tồn kho tổng) được cập nhật theo nhà cung cấp
```
