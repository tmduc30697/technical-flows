# Enhance sequence — Create session and route (virtual node + replicate N node)

Đây là **enhance**, cùng flow "create-session-and-route" đã có ở base nhưng nay thay đổi: ring có nhiều virtual node cho mỗi node vật lý nên tải phân bổ đều hơn, và session được ghi thành `SESSION_REPLICA` trên N node kế cận trên ring thay vì chỉ 1 node duy nhất. Đáp ứng yêu cầu 1 (virtual node) và đặt nền cho yêu cầu 3 (replication) của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant App as Login Service
    participant Ring as Hash Ring (nhiều virtual node/node)
    participant NodeA as NODE A (primary)
    participant NodeB as NODE B (replica kế tiếp)
    participant NodeC as NODE C (replica kế tiếp)

    User->>App: Đăng nhập thành công
    App->>Ring: Tra virtual node gần nhất cho hash(user_id)
    Ring-->>App: Virtual node thuộc NODE A, N node kế tiếp trên ring là NODE B và NODE C
    App->>NodeA: Ghi SESSION_REPLICA(role=primary, expires_at=now+3600s)
    par Replicate đồng thời sang N-1 node kế cận
        App->>NodeB: Ghi SESSION_REPLICA(role=replica, cùng expires_at)
        App->>NodeC: Ghi SESSION_REPLICA(role=replica, cùng expires_at)
    end
    NodeA-->>App: Xác nhận ghi primary thành công
    App-->>User: Trả session token
    Note over Ring: Vì mỗi node vật lý có nhiều virtual node trên ring, tải session mới được rải đều hơn thay vì dồn lệch vào vài node
```
