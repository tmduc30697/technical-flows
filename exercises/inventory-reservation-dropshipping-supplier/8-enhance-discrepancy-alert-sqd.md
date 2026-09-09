# Sequence - Enhance - Flow "discrepancy-alert"

Đây là **enhance**, flow hoàn toàn mới so với base, đáp ứng yêu cầu 5: theo dõi tỷ lệ hủy đơn do lệch tồn kho (`cancelled_out_of_stock`) theo từng nhà cung cấp, cảnh báo khi vượt ngưỡng để team vận hành biết đồng bộ dữ liệu với nhà cung cấp đó đang có vấn đề và có thể tạm ẩn sản phẩm của nhà cung cấp khỏi sàn.

```mermaid
sequenceDiagram
    participant App as Sàn dropshipping
    participant Metrics as Cancellation Metrics
    participant Alerting
    actor Ops as Team vận hành
    participant SupplierMgmt as Supplier Management

    loop Mỗi order kết thúc với status cancelled_out_of_stock
        App->>Metrics: ghi nhận 1 lần hủy do lệch tồn kho, gắn supplier_id
    end

    loop Theo mỗi khung thời gian (ví dụ mỗi giờ)
        Metrics-->>Metrics: tính cancel_rate = cancel_count / total_orders theo từng supplier_id
        Metrics->>Alerting: gửi cancel_rate hiện tại của từng nhà cung cấp

        alt cancel_rate vượt ngưỡng cấu hình
            Alerting-->>Alerting: ghi nhận supplier_discrepancy_alert, triggered_at = now
            Alerting->>Ops: cảnh báo, nhà cung cấp X có dấu hiệu lệch đồng bộ tồn kho
            Ops->>SupplierMgmt: tạm ẩn sản phẩm của nhà cung cấp X khỏi sàn
        else cancel_rate trong ngưỡng bình thường
            Alerting-->>Alerting: không cảnh báo
        end
    end
```
