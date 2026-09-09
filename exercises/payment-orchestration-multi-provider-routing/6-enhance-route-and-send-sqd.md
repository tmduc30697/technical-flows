# Enhance sequence — Chọn nhà cung cấp và gửi giao dịch (atomic lock)

Đây là **enhance**, flow "Định tuyến và gửi giao dịch tới nhà cung cấp" sau khi có update nguyên tử có điều kiện. So với base, flow này thay đổi ở chỗ chuyển trạng thái `pending → routing → sent_to_provider_A` bằng 1 câu update nguyên tử có điều kiện (`WHERE status = pending`), nên khi 2 worker cùng lấy nhầm 1 job, chỉ đúng 1 worker update thành công, worker còn lại nhận biết giao dịch đã bị worker khác xử lý và dừng lại — không gửi trùng request.

```mermaid
sequenceDiagram
    participant Worker1 as Routing Worker 1
    participant Worker2 as Routing Worker 2
    participant DB as TRANSACTION + PAYMENT_REQUEST store
    participant ProviderA as Provider A

    par Worker 1 lấy job từ hàng đợi routing
        Worker1->>DB: Đọc TRANSACTION #789 (status=pending), chọn Provider A theo tỷ lệ/phí/tình trạng sẵn sàng
        Worker1->>DB: UPDATE TRANSACTION SET status=sent_to_provider_A WHERE id=789 AND status=pending
    and Worker 2 cũng lấy đúng job này
        Worker2->>DB: Đọc TRANSACTION #789 (status=pending), chọn Provider A theo tỷ lệ/phí/tình trạng sẵn sàng
        Worker2->>DB: UPDATE TRANSACTION SET status=sent_to_provider_A WHERE id=789 AND status=pending
    end

    DB-->>Worker1: Update khớp 1 dòng, thành công
    Worker1->>DB: Ghi PAYMENT_REQUEST(provider=A, attempt=1), ROUTING_LOG(action=selected)
    Worker1->>ProviderA: Gửi request thanh toán kèm reference_code

    DB-->>Worker2: Update không khớp dòng nào (status đã đổi khỏi pending)
    Worker2->>Worker2: Nhận biết giao dịch đã được worker khác gán, dừng lại, không gửi request

    Note over Worker1,DB: Nhờ update nguyên tử có điều kiện, mỗi giao dịch chỉ được gán đúng 1 nhà cung cấp tại 1 thời điểm dù nhiều worker cùng đọc trùng job
```
