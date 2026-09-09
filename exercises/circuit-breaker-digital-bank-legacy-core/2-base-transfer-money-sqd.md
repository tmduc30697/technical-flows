# Base sequence — Transfer money (retry mù, không tra cứu trạng thái)

Đây là **base**, flow "Chuyển tiền" ở trạng thái hiện tại — timeout là retry ngay, không tra cứu xem core banking đã xử lý request trước đó chưa. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 2 và 5 của đề bài chính là sửa đúng lỗ hổng nguy hiểm này.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Digital Banking App
    participant Core as Core Banking (legacy)
    participant DB as TRANSACTION store

    Customer->>App: Xác nhận chuyển tiền
    App->>DB: Tạo TRANSACTION(status=pending)
    App->>Core: Gửi lệnh chuyển tiền
    Core-->>App: Timeout (không rõ core đã xử lý hay chưa — core rất chậm)
    App->>Core: Retry ngay lập tức, không tra cứu core_reference_id
    Core-->>App: Xử lý thành công (lần retry)
    Note over Core: Nếu lệnh đầu tiên thực ra đã được core xử lý trước khi timeout trả về, khách hàng bị chuyển tiền 2 lần
    App->>DB: Cập nhật TRANSACTION(status=success)
    App-->>Customer: "Chuyển tiền thành công"
    Note over App,Core: Không có log nào ghi lại việc đã retry, khó điều tra nếu khách hàng khiếu nại bị trừ tiền 2 lần
```
