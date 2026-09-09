# Base sequence — Nhận callback, map lại giao dịch theo số tiền + thời gian

Đây là **base**, flow "Nhận callback từ nhà cung cấp" ở trạng thái hiện tại — mỗi nhà cung cấp trả callback theo format riêng, hệ thống cố map ngược lại giao dịch gốc bằng cách so khớp số tiền và khoảng thời gian gần nhất, không dựa trên mã tham chiếu đã gửi kèm lúc khởi tạo. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 3 của đề bài nhằm sửa đúng nguy cơ map nhầm giao dịch khi trùng giá trị.

```mermaid
sequenceDiagram
    participant ProviderA as Provider A
    participant App as Orchestration Service
    participant DB as TRANSACTION + PAYMENT_REQUEST store

    ProviderA->>App: Callback format riêng của A (amount=500.000đ, result=OK, thời điểm gần đây)
    App->>DB: Tìm PAYMENT_REQUEST có amount=500.000đ, sent_at gần với thời điểm callback nhất

    Note over App,DB: Nếu cùng lúc có 2 giao dịch khác nhau đều gửi 500.000đ tới Provider A trong cùng khoảng thời gian ngắn, App có thể chọn nhầm giao dịch

    App->>DB: UPDATE PAYMENT_REQUEST tìm được SET status=success theo format nội bộ (mỗi provider tự map thủ công, chưa chuẩn hoá)
    App->>DB: UPDATE TRANSACTION SET status=success

    Note over App,DB: Không có mã tham chiếu gốc để đối chiếu chắc chắn, việc đoán theo số tiền + thời gian có thể gán nhầm callback cho giao dịch khác
```
