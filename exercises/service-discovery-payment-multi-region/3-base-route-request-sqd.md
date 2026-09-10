# Sequence Diagram — Base: Route Request

Đây là **base**, flow client discover instance qua registry và route request, đánh dấu instance unhealthy khi gọi thất bại — đây chính là flow sẽ được mở rộng thêm logic ưu tiên cùng region và fallback cross-region ở enhance.

```mermaid
sequenceDiagram
    actor Client as Client Service
    participant Registry as Service Registry
    participant InstanceA as Instance A
    participant InstanceB as Instance B

    Client->>Registry: Discover healthy instances of Service X
    Registry-->>Client: [Instance A, Instance B]
    Client->>InstanceA: Send request
    InstanceA--xClient: Timeout, no response
    Client->>Registry: Report failure for Instance A
    Registry->>Registry: Mark Instance A unhealthy after threshold
    Client->>InstanceB: Retry request on Instance B
    InstanceB-->>Client: Response OK
```
