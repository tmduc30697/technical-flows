# Base sequence — Chọn nhà cung cấp và gửi giao dịch (không lock atomic)

Đây là **base**, flow "Định tuyến và gửi giao dịch tới nhà cung cấp" ở trạng thái hiện tại — worker đọc trạng thái giao dịch, chọn nhà cung cấp theo tỷ lệ/phí/tình trạng sẵn sàng, rồi cập nhật trạng thái và gửi request, nhưng không dùng update nguyên tử có điều kiện khi chuyển trạng thái. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 2 của đề bài (mỗi giao dịch chỉ được gán đúng 1 nhà cung cấp) nhằm sửa đúng lỗ hổng race giữa 2 worker ở đây.

```mermaid
sequenceDiagram
    participant Worker1 as Routing Worker 1
    participant Worker2 as Routing Worker 2
    participant DB as TRANSACTION + PAYMENT_REQUEST store
    participant ProviderA as Provider A

    par Worker 1 lấy job từ hàng đợi routing
        Worker1->>DB: Đọc TRANSACTION #789 (status=pending)
        Worker1->>Worker1: Chọn Provider A theo tỷ lệ/phí/tình trạng sẵn sàng
    and Worker 2 cũng lấy đúng job này (do lỗi đọc trùng hàng đợi)
        Worker2->>DB: Đọc TRANSACTION #789 (status=pending, chưa thấy update của Worker 1)
        Worker2->>Worker2: Chọn Provider A theo tỷ lệ/phí/tình trạng sẵn sàng
    end

    Worker1->>DB: UPDATE TRANSACTION SET status=sent_to_provider, ghi PAYMENT_REQUEST tới Provider A
    Worker1->>ProviderA: Gửi request thanh toán

    Worker2->>DB: UPDATE TRANSACTION SET status=sent_to_provider, ghi PAYMENT_REQUEST tới Provider A (ghi đè, không biết Worker 1 đã gửi)
    Worker2->>ProviderA: Gửi lại request thanh toán y hệt

    Note over ProviderA,DB: Không có cơ chế khoá atomic nên 2 worker có thể cùng gán và cùng gửi request cho 1 giao dịch, dẫn tới nguy cơ charge tiền khách 2 lần
```
