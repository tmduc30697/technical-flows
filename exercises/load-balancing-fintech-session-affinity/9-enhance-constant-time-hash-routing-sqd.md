# Enhance sequence — Constant-time hash routing

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base không có bước hash nào nên không phát sinh rủi ro timing attack. Đáp ứng **yêu cầu 5** (đảm bảo hash key không lộ thông tin nhạy cảm qua timing attack, thời gian route không được khác biệt đáng kể giữa các user).

```mermaid
sequenceDiagram
    actor UserX as User X (transaction_id thông thường)
    actor UserY as User Y (transaction_id có pattern đặc biệt)
    participant LB as Load Balancer
    participant Ring as HASH_RING
    participant Timing as ROUTING_TIMING_SAMPLE

    UserX->>LB: Request route theo transaction_id của User X
    LB->>Ring: Hash bằng HMAC-SHA256 (constant-time), tìm instance trên ring bằng thuật toán tra cứu độ phức tạp cố định
    Ring-->>LB: Trả về instance, thời gian xử lý gần như không đổi bất kể giá trị input
    LB->>Timing: Ghi ROUTING_TIMING_SAMPLE(routing_duration_ms)

    UserY->>LB: Request route theo transaction_id của User Y
    LB->>Ring: Hash bằng HMAC-SHA256 (constant-time), tra cứu cùng thuật toán
    Ring-->>LB: Trả về instance, thời gian xử lý tương đương User X
    LB->>Timing: Ghi ROUTING_TIMING_SAMPLE(routing_duration_ms)

    Timing->>Timing: So sánh routing_duration_ms giữa nhiều user, xác nhận không có khác biệt đáng kể
    Note over LB,Ring: Dùng HMAC với khóa bí mật thay vì hash trần lộ transaction_id, cộng với tra cứu ring có độ phức tạp cố định, giúp không ai suy luận được thông tin nhạy cảm qua việc đo thời gian route
```
