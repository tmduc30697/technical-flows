# Enhance sequence — Transfer money (tra cứu trạng thái trước khi retry)

Đây là **enhance**, cùng flow "Transfer money" đã có ở base nhưng nay thay đổi hoàn toàn theo yêu cầu 2 và 5 của đề bài: chỉ retry khi chắc chắn core banking chưa xử lý, dựa vào tra cứu core_reference_id, và ghi log đầy đủ mọi quyết định.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Digital Banking App
    participant Core as Core Banking (legacy)
    participant DB as TRANSACTION store
    participant Audit as RETRY_AUDIT_LOG store

    Customer->>App: Xác nhận chuyển tiền
    App->>DB: Tạo TRANSACTION(status=pending, core_reference_id)
    App->>Core: Gửi lệnh chuyển tiền kèm core_reference_id
    Core-->>App: Timeout (không rõ đã xử lý hay chưa)
    App->>Audit: Ghi RETRY_AUDIT_LOG(action=checked_status_before_retry)
    App->>Core: Tra cứu trạng thái theo core_reference_id (API core hỗ trợ tra cứu)
    Core-->>App: Trả trạng thái thực tế của request trước đó

    alt Core xác nhận CHƯA xử lý request trước
        App->>Audit: Ghi RETRY_AUDIT_LOG(action=retried, result=safe)
        App->>Core: Retry lệnh chuyển tiền
        Core-->>App: Xử lý thành công
        App->>DB: Cập nhật TRANSACTION(status=success)
        App-->>Customer: "Chuyển tiền thành công"
    else Core xác nhận ĐÃ xử lý request trước (thành công hoặc thất bại)
        App->>Audit: Ghi RETRY_AUDIT_LOG(action=skipped_retry, result=already_processed)
        App->>DB: Cập nhật TRANSACTION theo đúng trạng thái core đã xác nhận, không gửi lệnh mới
        App-->>Customer: Kết quả chính xác, không có nguy cơ chuyển tiền 2 lần
    end
```
