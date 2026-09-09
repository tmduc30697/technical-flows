# Enhance sequence — Dashboard tỷ lệ thành công real-time và log lịch sử routing

Đây là **enhance**, flow mới phát sinh từ enhance — tổng hợp `PROVIDER_METRIC` real-time theo từng nhà cung cấp để tự động điều chỉnh `routing_weight`, và cho phép tra cứu toàn bộ `ROUTING_LOG` của 1 giao dịch (đã thử nhà cung cấp nào, theo thứ tự nào, kết quả gì) phục vụ điều tra khi có khiếu nại double-charge như tình huống ở file `9-enhance-race-failover-vs-callback-sqd.md`.

```mermaid
sequenceDiagram
    participant App as Orchestration Service
    participant Metric as PROVIDER_METRIC store
    participant Dashboard as Routing Dashboard
    participant RoutingEngine as Routing Weight Adjuster
    participant Support as Đội hỗ trợ khách hàng
    participant Log as ROUTING_LOG store

    par Mỗi khi 1 PAYMENT_REQUEST kết thúc (success/failed)
        App->>Metric: Cộng dồn success_count/failure_count vào PROVIDER_METRIC của provider tương ứng theo window hiện tại
    end

    Dashboard->>Metric: Đọc PROVIDER_METRIC real-time theo từng provider
    Dashboard->>Dashboard: Hiển thị tỷ lệ thành công/lỗi real-time theo từng nhà cung cấp

    RoutingEngine->>Metric: Đọc success_rate gần nhất theo từng provider
    RoutingEngine->>App: Tự động điều chỉnh routing_weight (ưu tiên provider có tỷ lệ thành công cao hơn)

    Support->>Log: Khách khiếu nại double-charge giao dịch #789, tra cứu ROUTING_LOG theo transaction_id
    Log-->>Support: Toàn bộ lịch sử: selected A → sent A → timeout → verified_failed → failover B → sent B → success B → callback A muộn → cancelled_due_to_double_success A
    Support->>Support: Dùng log này xác nhận đã hoàn tiền đúng phía A, kết luận khiếu nại
```
