# Base sequence — Check rate limit (routing table không đồng bộ giữa router)

Đây là **base**, flow kiểm tra rate limit cho 1 request. Vì mỗi router instance giữ bản `ROUTING_TABLE` cục bộ riêng và có thể lệch nhau (chưa kịp đồng bộ sau lần cập nhật gần nhất), 2 request của cùng 1 API key qua 2 router khác nhau có thể bị route tới 2 node khác nhau. Đây chính là vấn đề yêu cầu 1 của đề bài mô tả.

```mermaid
sequenceDiagram
    actor ClientApp as Client gọi API
    participant Router1 as ROUTER_INSTANCE 1 (bảng cũ)
    participant Router2 as ROUTER_INSTANCE 2 (bảng đã cập nhật)
    participant NodeA as NODE A
    participant NodeB as NODE B

    ClientApp->>Router1: Request 1 dùng API key K (limit 100/phút)
    Router1->>Router1: Tra ROUTING_TABLE cục bộ (bản cũ), K thuộc NODE A
    Router1->>NodeA: Forward request, tăng RATE_LIMIT_COUNTER(K) lên 80/100

    ClientApp->>Router2: Request 2 cũng dùng API key K, qua router khác
    Router2->>Router2: Tra ROUTING_TABLE cục bộ (bản mới hơn, sau khi cụm đổi cấu hình), K thuộc NODE B
    Router2->>NodeB: Forward request, NODE B chưa từng thấy key K nên khởi tạo RATE_LIMIT_COUNTER(K)=1/100

    Note over NodeA,NodeB: Cùng 1 API key K nhưng bị đếm độc lập trên 2 node khác nhau, tổng số request thực tế cho phép vượt xa giới hạn 100/phút thật sự
```
