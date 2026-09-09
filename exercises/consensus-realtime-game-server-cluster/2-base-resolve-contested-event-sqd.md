# Base sequence — Resolve contested event (dựa vào client_timestamp)

Đây là **base**, flow xử lý 2 sự kiện tranh chấp gần như đồng thời (2 người chơi cùng bắn trúng 1 mục tiêu). Ở base, node authoritative sắp thứ tự dựa trên `client_timestamp` do từng client tự gửi kèm — đây chính là vấn đề mà yêu cầu 1 của đề bài chỉ ra (client có độ trễ mạng khác nhau nên timestamp không đáng tin), làm nền để so sánh với cách giải quyết bằng log replicate qua majority ở enhance.

```mermaid
sequenceDiagram
    actor ClientA as Client A
    actor ClientB as Client B
    participant Node as Authoritative NODE
    participant State as MATCH_STATE

    Note over ClientA,ClientB: Cả 2 người chơi bắn trúng mục tiêu gần như cùng lúc, thực tế A bắn trước B vài ms
    ClientA->>Node: INPUT_EVENT(event_type=hit, client_timestamp=t1)
    ClientB->>Node: INPUT_EVENT(event_type=hit, client_timestamp=t2)
    Note over Node: Do độ trễ mạng khác nhau, Node nhận input của B trước input của A dù A bắn trước
    Node->>Node: Sắp thứ tự theo client_timestamp (t2 < t1, do đồng hồ và độ trễ của B lệch)
    Node->>State: Áp dụng kết quả, B được tính là người bắn trúng trước
    Node-->>ClientA: Broadcast kết quả, "B thắng tranh chấp"
    Node-->>ClientB: Broadcast kết quả, "B thắng tranh chấp"
    Note over ClientA: A cảm thấy vô lý vì trải nghiệm cục bộ của A thấy mình bắn trước, kết quả không đáng tin do dựa vào timestamp client
```
