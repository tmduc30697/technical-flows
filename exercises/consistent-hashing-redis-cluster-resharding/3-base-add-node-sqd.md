# Base sequence — Add node (rehash toàn bộ, cache-miss storm)

Đây là **base**, flow thêm 1 node vào cụm cache đang dùng hashing modulo N. Vì N đổi thành N+1, công thức `hash(key) mod N` cho ra kết quả khác cho gần như mọi key, dù key đó không thực sự cần đổi node — toàn bộ dữ liệu coi như phải rehash lại, gây cache-miss hàng loạt. Đây chính là vấn đề mà yêu cầu 1 và 2 của đề bài muốn giải quyết bằng consistent hashing.

```mermaid
sequenceDiagram
    actor Service as Service gọi cache
    participant Cluster as Cache Cluster
    participant NodeA as NODE A (cũ, giữ K1)
    participant NodeNew as NODE mới thêm vào

    Cluster->>Cluster: Thêm NODE mới, N tăng lên N+1
    Note over Cluster: Công thức hash(key) mod N đổi hoàn toàn khi N đổi, kể cả với key không liên quan gì tới node mới

    Service->>Cluster: GET key=K1 (trước đó đã set ở NODE A)
    Cluster->>Cluster: Tính lại hash(K1) mod (N+1), ra NODE mới (không phải NODE A nữa)
    Cluster->>NodeNew: Đọc CACHE_ENTRY(K1)
    NodeNew-->>Cluster: Không có (dữ liệu thực tế vẫn nằm ở NODE A, chưa được di chuyển)
    Cluster-->>Service: Cache miss

    Note over NodeA,NodeNew: Hầu hết key trong cụm rơi vào tình huống tương tự K1, gây cache-miss storm đồng loạt, dồn toàn bộ traffic xuống database gốc ngay sau khi thêm 1 node
```
