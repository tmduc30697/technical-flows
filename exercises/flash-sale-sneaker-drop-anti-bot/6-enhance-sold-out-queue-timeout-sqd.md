# Enhance sequence — Hàng đợi hết hàng, trả kết quả trong thời gian giới hạn

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 4** của đề bài: khi tồn kho giảm về 0, mọi request đang chờ trong `QUEUE_TICKET` phải được trả kết quả "hết hàng" trong một khoảng thời gian giới hạn rõ ràng, không để user chờ vô thời hạn.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Checkout Service
    participant Queue as Queue Worker
    participant DB as EVENT/QUEUE_TICKET store
    participant Sweeper as Timeout Sweeper (chạy định kỳ)

    Customer->>App: Gửi request mua (đã qua challenge)
    App->>DB: INSERT QUEUE_TICKET (status=queued, expires_at=now+15s)
    App-->>Customer: "Đang xử lý, vui lòng chờ" (kèm ticket_id)

    Queue->>DB: Lấy vé kế tiếp theo thứ tự FIFO
    Queue->>DB: UPDATE EVENT SET remaining_stock=remaining_stock-1 WHERE remaining_stock>0
    DB-->>Queue: 0 dòng bị ảnh hưởng (remaining_stock đã = 0)
    Queue->>DB: UPDATE QUEUE_TICKET SET status=out_of_stock

    loop Mỗi 2 giây cho tới khi có kết quả hoặc hết 15 giây
        Customer->>App: Poll GET /queue-ticket/{ticket_id}
        App->>DB: SELECT status FROM QUEUE_TICKET WHERE id=ticket_id
        DB-->>App: status=out_of_stock
        App-->>Customer: "Rất tiếc, đã hết hàng"
    end

    Note over Sweeper,DB: Với các vé còn status=queued mà đã quá expires_at (ví dụ do bug hoặc worker chậm), sweeper chủ động đánh dấu out_of_stock để đảm bảo không có vé nào treo quá 15 giây
```
