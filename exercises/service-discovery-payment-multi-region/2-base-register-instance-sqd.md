# Sequence Diagram — Base: Register Instance

Đây là **base**, flow một instance tự đăng ký vào registry khi khởi động và định kỳ được health check — tiền đề bắt buộc để sau này registry có dữ liệu instance mà gắn thêm metadata region.

```mermaid
sequenceDiagram
    participant Instance as Service Instance
    participant Registry as Service Registry

    Instance->>Registry: Register (service_id, host, port)
    Registry->>Registry: Create INSTANCE record (status=healthy)
    Registry-->>Instance: Registration confirmed

    loop periodic
        Registry->>Instance: Health check probe
        Instance-->>Registry: OK
    end
```
