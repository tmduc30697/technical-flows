# Base sequence — Query metrics theo khoảng thời gian (phải scan lẫn dữ liệu service khác)

Đây là **base**, flow truy vấn metrics trong 1 khoảng thời gian của 1 service cụ thể. Vì partition base chỉ chia theo thời gian, dữ liệu của mọi service bị trộn lẫn trong cùng partition, nên dù đã xác định đúng các time bucket liên quan, hệ thống vẫn phải scan toàn bộ dữ liệu trong đó rồi lọc theo `service_id`. Flow này là tiền đề cho yêu cầu 2 của đề bài (truy vấn phải chỉ chạm partition tối thiểu và không cần scan dư thừa).

```mermaid
sequenceDiagram
    actor App as Dashboard App
    participant Router as Query Router
    participant P1 as PARTITION 09:00-10:00
    participant P2 as PARTITION 10:00-11:00

    App->>Router: Lấy metrics 1 giờ qua của Service A
    Router->>Router: Xác định các time_bucket rơi vào khoảng đó (09:00-10:00, 10:00-11:00)
    Router->>P1: Scan toàn bộ điểm dữ liệu trong partition (mọi service)
    P1-->>Router: Trả về dữ liệu lẫn lộn nhiều service
    Router->>P2: Scan toàn bộ điểm dữ liệu trong partition (mọi service)
    P2-->>Router: Trả về dữ liệu lẫn lộn nhiều service
    Router->>Router: Lọc lại chỉ giữ điểm dữ liệu của Service A
    Router-->>App: Kết quả (đã lọc)
    Note over Router: Chi phí đọc và băng thông I/O bị lãng phí đáng kể vì phải quét cả dữ liệu của các service không liên quan trong cùng partition
```
