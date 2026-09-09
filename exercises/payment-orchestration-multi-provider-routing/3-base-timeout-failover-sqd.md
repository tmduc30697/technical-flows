# Base sequence — Timeout ở nhà cung cấp A, failover ngay sang B (không xác minh)

Đây là **base**, flow "Timeout khi gửi thanh toán, chuyển sang nhà cung cấp khác" ở trạng thái hiện tại — khi request tới Provider A timeout, hệ thống coi ngay là thất bại và gửi request y hệt sang Provider B mà không xác minh lại trạng thái thật ở A. Flow này liên quan mật thiết tới enhance vì yêu cầu đầu tiên của đề bài nhằm sửa đúng nguy cơ double-charge do failover mù quáng này.

```mermaid
sequenceDiagram
    participant App as Orchestration Service
    participant ProviderA as Provider A
    participant ProviderB as Provider B
    participant DB as PAYMENT_REQUEST store

    App->>ProviderA: Gửi request thanh toán giao dịch #789
    ProviderA-->>App: Timeout (không rõ đã charge hay chưa ở phía A)
    App->>DB: Ghi PAYMENT_REQUEST(provider=A, status=timeout)

    Note over App: Coi timeout = thất bại ngay lập tức, không query lại trạng thái ở A, không đợi đủ thời gian xác định

    App->>ProviderB: Gửi luôn request thanh toán y hệt giao dịch #789
    ProviderB-->>App: Thành công
    App->>DB: Ghi PAYMENT_REQUEST(provider=B, status=success)

    Note over ProviderA,DB: Nếu thực tế request đầu tiên tới A đã kịp xử lý charge thành công trước khi timeout trả về, khách hàng đã bị charge ở cả A và B
```
