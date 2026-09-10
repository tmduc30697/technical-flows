# Sequence Diagram — Enhance: Route Request

Đây là **enhance**, flow route request đã đổi tên vẫn giữ nguyên nhưng logic thay đổi hoàn toàn so với base: client giờ ưu tiên tìm instance healthy trong cùng region, chỉ fallback sang region khác khi region hiện tại hết instance healthy, đồng thời request bị route cross-region phải được đánh dấu (header/log) và tạo alert cho ops.

```mermaid
sequenceDiagram
    actor Client as Client Service (Region A)
    participant RegistryA as Registry (Region A)
    participant InstB as Instance (Region B)
    participant Alert as Alert Service

    Client->>RegistryA: Discover healthy instances of Service X in Region A
    RegistryA-->>Client: No healthy instance found in Region A
    Note over Client,RegistryA: So với base, client không dừng ở đây mà tiếp tục fallback sang region khác

    Client->>RegistryA: Query synced cross-region view for Service X
    RegistryA-->>Client: Region B currently has healthy instances
    Client->>InstB: Send request, header X-Cross-Region-Failover=true
    InstB-->>Client: Response OK, marked as cross-region

    Client->>RegistryA: Report FAILOVER_EVENT (service, from=A, to=B, reason)
    RegistryA->>Alert: Trigger cross-region failover ALERT
    Alert-->>Client: Ops team notified

    Note over Client,InstB: Mọi request bị route sang region khác được đánh dấu header/log để debug độ trễ tăng bất thường
```
