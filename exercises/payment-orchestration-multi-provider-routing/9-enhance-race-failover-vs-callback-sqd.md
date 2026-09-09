# Enhance sequence — Race giữa callback thất bại thật từ A và failover đã chuyển sang B

Đây là **enhance**, flow mới phát sinh từ enhance — minh hoạ đúng tình huống mô tả cụ thể ở yêu cầu thứ 4: callback thật từ Provider A báo thất bại đến ngay sau khi hệ thống đã quyết định failover sang Provider B do quá thời gian chờ xác minh, và cả 2 vô tình đều thành công. Đảm bảo chỉ 1 trong 2 `PAYMENT_REQUEST` được coi là hợp lệ cuối cùng, phía còn lại bị huỷ/hoàn tiền.

```mermaid
sequenceDiagram
    participant ProviderA as Provider A
    participant App as Orchestration Service
    participant ProviderB as Provider B
    participant DB as PAYMENT_REQUEST + TRANSACTION store

    Note over App,DB: Request tới A đã timeout, App đợi quá thời gian xác minh cho phép mà chưa có phản hồi rõ ràng, quyết định failover sang B (theo flow 7-enhance-timeout-failover-sqd.md)

    App->>DB: UPDATE PAYMENT_REQUEST(provider=A) SET status=failed (do hết thời gian chờ xác minh), ghi ROUTING_LOG(action=failover)
    App->>ProviderB: Gửi request thanh toán (reference_code=REF-789, attempt=2)
    ProviderB-->>App: Thành công
    App->>DB: UPDATE PAYMENT_REQUEST(provider=B) SET status=success, UPDATE TRANSACTION SET status=success

    Note over ProviderA,App: Ngay sau đó, callback thật từ A đến muộn, báo rằng request đầu tiên thực ra cũng đã charge thành công

    ProviderA->>App: Callback (reference_code=REF-789, normalized_status=success)
    App->>DB: Đọc TRANSACTION #789, thấy đã có PAYMENT_REQUEST(provider=B) status=success là bản ghi hợp lệ hiện tại

    App->>App: Áp quy tắc chỉ 1 PAYMENT_REQUEST được coi hợp lệ — ưu tiên giữ bản ghi đã được xác nhận thành công trước (provider=B), coi A là double-success ngoài ý muốn
    App->>DB: UPDATE PAYMENT_REQUEST(provider=A) SET status=cancelled_refunded, ghi ROUTING_LOG(action=cancelled_due_to_double_success)
    App->>ProviderA: Gọi API huỷ/hoàn tiền cho giao dịch đã lỡ charge ở A
    ProviderA-->>App: Xác nhận đã hoàn tiền

    Note over App,DB: TRANSACTION #789 chỉ được tính thành công đúng 1 lần dù cả A và B đều vô tình charge thành công, tránh double-charge tới khách hàng
```
