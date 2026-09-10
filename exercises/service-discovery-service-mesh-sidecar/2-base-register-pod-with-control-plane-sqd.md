# Sequence Diagram — Base: Register Pod With Control Plane

Đây là **base**, flow pod mới được scheduler đưa lên, sidecar tự discover danh sách service khác qua control plane — tiền đề bắt buộc để sau này bổ sung cơ chế warm-up trước khi nhận traffic và cache khi control plane không phản hồi.

```mermaid
sequenceDiagram
    participant Scheduler as K8s Scheduler
    participant Pod as Pod mới
    participant Sidecar as Sidecar
    participant ControlPlane as Control Plane

    Scheduler->>Pod: Schedule pod lên node
    Pod->>Sidecar: Khởi động cùng pod
    Sidecar->>ControlPlane: Đăng ký + lấy danh sách service/endpoint
    ControlPlane-->>Sidecar: Danh sách endpoint hiện tại
    Sidecar->>Sidecar: Bắt đầu nhận traffic ngay
```
