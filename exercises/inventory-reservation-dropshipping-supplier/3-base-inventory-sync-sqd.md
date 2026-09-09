# Sequence - Base - Flow "inventory-sync"

Đây là **base**: đồng bộ tồn kho định kỳ từ nhà cung cấp về cache nội bộ bằng cách ghi đè trực tiếp giá trị mới nhất. Flow này được chọn vì nó là tiền đề cho yêu cầu 3 - ghi đè đơn giản như thế này sẽ xóa mất mọi thay đổi tạm thời (ví dụ order đang được xử lý song song) diễn ra trong lúc đồng bộ chạy.

```mermaid
sequenceDiagram
    participant Scheduler as Sync Scheduler
    participant SupplierAPI as API Nhà cung cấp
    participant Cache as Inventory Cache

    loop Mỗi vài phút
        Scheduler->>SupplierAPI: lấy tồn kho mới nhất cho sản phẩm X
        SupplierAPI-->>Scheduler: quantity thật = 3

        Note over Scheduler,Cache: Trong lúc này có thể có order khác đang trừ tạm cache song song

        Scheduler->>Cache: ghi đè cached_quantity = 3
        Note over Cache: Ghi đè toàn bộ, không xét order đang xử lý song song,\ncó thể làm mất thay đổi tạm thời vừa được trừ
    end
```
