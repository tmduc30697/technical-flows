# Enhance sequence — Post-sale reconciliation

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đáp ứng yêu cầu thứ 4 của đề bài: đối soát tổng số bán ra theo ghi nhận hệ thống với tồn kho vật lý thực tế, có báo cáo chênh lệch nếu có.

```mermaid
sequenceDiagram
    actor Ops as Ops/Kho vận
    participant Job as Reconciliation Job
    participant Nodes as INVENTORY_NODE (tất cả replica)
    participant OrderDB as Order/Sales store
    participant Report as RECONCILIATION_REPORT store

    Note over Job: Sau khi flash sale kết thúc
    Job->>OrderDB: Tổng hợp total_sold_recorded từ toàn bộ đơn hàng thành công
    Job->>Nodes: Lấy stock_value còn lại từ toàn bộ replica đã hội tụ (sau khi partition nếu có đã hàn lại)
    Ops->>Job: Cung cấp physical_stock_actual (kiểm kho vật lý thực tế)

    Job->>Job: Tính discrepancy = (initial_stock - total_sold_recorded) - physical_stock_actual
    alt discrepancy = 0
        Job->>Report: Ghi RECONCILIATION_REPORT (discrepancy=0)
        Report-->>Ops: "Khớp hoàn toàn, không có chênh lệch"
    else discrepancy khác 0
        Job->>Report: Ghi RECONCILIATION_REPORT kèm discrepancy và chi tiết
        Report-->>Ops: Báo cáo chênh lệch để điều tra (vd oversell do minority từng lọt qua, hoặc lỗi ghi nhận đơn hàng)
    end
```
