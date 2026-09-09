# Enhance sequence — Check rate limit (tra chung 1 hash ring duy nhất)

Đây là **enhance**, cùng flow "check-rate-limit" đã có ở base nhưng nay thay đổi: mọi router instance tra chung `HASH_RING` (nguồn sự thật duy nhất, có version rõ ràng) thay vì bảng cục bộ riêng, nên luôn route 1 API key về đúng 1 node. Đáp ứng yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    actor ClientApp as Client gọi API
    participant Router1 as ROUTER_INSTANCE 1
    participant Router2 as ROUTER_INSTANCE 2
    participant Ring as HASH_RING (dùng chung)
    participant NodeA as NODE A

    ClientApp->>Router1: Request 1 dùng API key K
    Router1->>Ring: Tra shard cho hash(K), kèm ring_version hiện tại
    Ring-->>Router1: K thuộc NODE A
    Router1->>NodeA: Forward request, tăng RATE_LIMIT_COUNTER(K) lên 80/100

    ClientApp->>Router2: Request 2 cũng dùng API key K, qua router khác
    Router2->>Ring: Tra shard cho hash(K), cùng dùng chung ring nên luôn ra cùng kết quả
    Ring-->>Router2: K thuộc NODE A (giống hệt lần tra trước)
    Router2->>NodeA: Forward request, tăng RATE_LIMIT_COUNTER(K) lên 81/100

    Note over Ring,NodeA: Vì không còn bảng cục bộ riêng theo từng router, không thể có 2 router route cùng 1 key về 2 node khác nhau, độ trễ thêm chỉ là 1 lần tra ring
```
