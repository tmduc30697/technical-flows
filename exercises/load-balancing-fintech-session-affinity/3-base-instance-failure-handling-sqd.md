# Base sequence — Instance failure handling

Đây là **base**, flow "Xử lý khi instance chết giữa giao dịch" ở trạng thái hiện tại: khi instance đang giữ state trong bộ nhớ chết đột ngột, không có cơ chế phát hiện hay khôi phục nào, giao dịch bị treo. Flow này là tiền đề cho **yêu cầu 2** (phát hiện và fail-over kèm khôi phục state từ nguồn bền) vì đây chính là hậu quả cần khắc phục.

```mermaid
sequenceDiagram
    actor Customer
    participant LB as Load Balancer
    participant InstanceA as Instance A (đang giữ state OTP trong bộ nhớ)

    Customer->>LB: Bước 1, khởi tạo giao dịch (transaction_id=T2)
    LB->>InstanceA: Route round-robin
    InstanceA->>InstanceA: Sinh OTP, lưu trong bộ nhớ (không backup ra nơi bền vững)
    InstanceA-->>Customer: Yêu cầu nhập OTP

    Note over InstanceA: Instance A gặp sự cố, tiến trình chết đột ngột, toàn bộ state trong bộ nhớ bị mất

    Customer->>LB: Bước 2, gửi OTP xác nhận
    LB--xInstanceA: Không route được vì instance đã chết
    Note over LB,Customer: Không có cơ chế phát hiện instance chết giữa giao dịch để chuyển sang instance khác kèm khôi phục state, giao dịch bị treo vô thời hạn
```
