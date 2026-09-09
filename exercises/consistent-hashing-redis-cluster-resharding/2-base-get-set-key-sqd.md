# Base sequence — Get/set key (hash modulo N node)

Đây là **base**, flow đọc/ghi 1 key vào cache cluster ở trạng thái hiện tại — node phụ trách được xác định bằng `hash(key) mod N`. Cách này hoạt động bình thường khi cụm ổn định, nhưng là nguyên nhân trực tiếp gây ra vấn đề khi N thay đổi (xem flow "add-node"), làm nền cho toàn bộ đề bài về resharding.

```mermaid
sequenceDiagram
    actor Service as Service gọi cache
    participant Cluster as Cache Cluster (N node)
    participant NodeA as NODE A

    Service->>Cluster: SET key=K1, value=V1
    Cluster->>Cluster: Tính hash(K1) mod N, ra NODE A
    Cluster->>NodeA: Ghi CACHE_ENTRY(K1, V1)
    NodeA-->>Service: OK

    Service->>Cluster: GET key=K1
    Cluster->>Cluster: Tính hash(K1) mod N, vẫn ra NODE A (N chưa đổi)
    Cluster->>NodeA: Đọc CACHE_ENTRY(K1)
    NodeA-->>Service: V1
```
