# Enhance sequence — Timeout ở A: xác minh trạng thái trước khi failover sang B

Đây là **enhance**, flow "Timeout khi gửi thanh toán, chuyển sang nhà cung cấp khác" sau khi có bước xác minh bắt buộc. So với base, flow này thay đổi ở chỗ khi request tới Provider A timeout, hệ thống **không** gửi ngay request y hệt sang Provider B — mà trước tiên gọi API query status của A, hoặc đợi đủ khoảng thời gian timeout xác định trước, rồi mới coi là thất bại thật và failover, tránh nguy cơ charge tiền khách ở cả A và B.

```mermaid
sequenceDiagram
    participant App as Orchestration Service
    participant ProviderA as Provider A
    participant ProviderB as Provider B
    participant DB as PAYMENT_REQUEST + ROUTING_LOG store

    App->>ProviderA: Gửi request thanh toán giao dịch #789 (reference_code=REF-789)
    ProviderA-->>App: Timeout (không rõ đã charge hay chưa ở phía A)
    App->>DB: UPDATE PAYMENT_REQUEST(provider=A) SET status=verifying, ghi ROUTING_LOG(action=timeout)

    App->>ProviderA: Query status giao dịch theo reference_code=REF-789

    alt Provider A xác nhận chưa xử lý / đã thất bại thật
        ProviderA-->>App: status=not_found hoặc failed
        App->>DB: UPDATE PAYMENT_REQUEST(provider=A) SET status=failed, ghi ROUTING_LOG(action=verified_failed)
        App->>DB: UPDATE TRANSACTION SET status=routing (chuẩn bị failover)
        App->>DB: Ghi PAYMENT_REQUEST mới (provider=B, attempt=2), ROUTING_LOG(action=failover)
        App->>ProviderB: Gửi request thanh toán kèm cùng reference_code=REF-789
        ProviderB-->>App: Thành công
        App->>DB: UPDATE PAYMENT_REQUEST(provider=B) SET status=success
    else Provider A xác nhận đã charge thành công (dù response ban đầu timeout)
        ProviderA-->>App: status=success
        App->>DB: UPDATE PAYMENT_REQUEST(provider=A) SET status=success, KHÔNG failover sang B
        Note over App,ProviderB: Tránh được double-charge vì đã xác minh trước khi quyết định chuyển hướng
    end
```
