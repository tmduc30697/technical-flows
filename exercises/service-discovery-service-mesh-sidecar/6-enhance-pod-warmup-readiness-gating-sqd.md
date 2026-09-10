# Sequence Diagram — Enhance: Pod Warmup Readiness Gating

Đây là **enhance**, flow hoàn toàn mới cho pod vừa được scheduler đưa lên do autoscale — pod không được nhận traffic ngay như ở base, mà phải chờ readiness check pass ít nhất N lần liên tiếp để tránh nhận request trong lúc đang warm-up (load config, kết nối DB pool...).

```mermaid
sequenceDiagram
    participant Scheduler as K8s Scheduler
    participant Pod as Pod mới (autoscale)
    participant Sidecar as Sidecar
    participant ControlPlane as Control Plane

    Scheduler->>Pod: Schedule pod lên node (autoscale)
    Pod->>Sidecar: Khởi động cùng pod
    Sidecar->>ControlPlane: Đăng ký endpoint, nhưng đánh dấu not_ready
    Note over Sidecar,ControlPlane: Chưa đưa vào routing table cho các sidecar khác

    loop cho tới khi đủ N lần pass liên tiếp
        Sidecar->>Pod: Readiness check
        alt pass
            Pod-->>Sidecar: OK
            Sidecar->>Sidecar: Tăng readiness_consecutive_pass
        else fail
            Pod--xSidecar: Chưa sẵn sàng (đang load config/DB pool)
            Sidecar->>Sidecar: Reset readiness_consecutive_pass về 0
        end
    end

    Sidecar->>ControlPlane: readiness_consecutive_pass đạt N, cập nhật readiness_status=ready
    ControlPlane->>ControlPlane: Đưa endpoint vào danh sách route cho các sidecar khác
```
