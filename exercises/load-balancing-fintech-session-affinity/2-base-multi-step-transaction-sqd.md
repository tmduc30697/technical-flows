# Base sequence — Multi-step transaction (OTP)

Đây là **base**, flow "Giao dịch nhiều bước" ở trạng thái hiện tại: mỗi bước của cùng một giao dịch được LB route round-robin độc lập, không có gì đảm bảo hai bước liên tiếp rơi vào cùng instance. Flow này là tiền đề cho **yêu cầu 1** (consistent hashing đảm bảo cùng transaction_id luôn map tới cùng instance) vì đây chính là lỗ hổng mà consistent hashing sẽ khắc phục.

```mermaid
sequenceDiagram
    actor Customer
    participant LB as Load Balancer (round-robin)
    participant InstanceA as Instance A
    participant InstanceB as Instance B

    Customer->>LB: Bước 1, gửi request khởi tạo chuyển tiền (transaction_id=T1)
    LB->>InstanceA: Route theo round-robin
    InstanceA->>InstanceA: Sinh OTP, lưu OTP tạm trong bộ nhớ instance (TRANSACTION_STATE_CACHE)
    InstanceA-->>Customer: Yêu cầu nhập OTP

    Customer->>LB: Bước 2, gửi OTP xác nhận (cùng transaction_id=T1)
    LB->>InstanceB: Route theo round-robin (rơi vào instance khác vòng quay)
    InstanceB->>InstanceB: Không có OTP nào trong bộ nhớ vì OTP được sinh ở InstanceA
    InstanceB-->>Customer: Lỗi "không tìm thấy giao dịch" hoặc phải đọc lại toàn bộ state từ DB, tốn thêm một round-trip
    Note over InstanceA,InstanceB: Vì không có session affinity, mỗi bước có thể rơi vào instance khác, buộc phải đọc lại state từ DB mỗi bước hoặc lỗi hẳn
```
