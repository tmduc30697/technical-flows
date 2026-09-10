# Sequence Diagram — Enhance: Control Plane Outage Fallback

Đây là **enhance**, flow hoàn toàn mới xử lý khi control plane tạm thời không phản hồi — sidecar phải dùng `ENDPOINT_CACHE_SNAPSHOT` gần nhất để tiếp tục route traffic (fail open có kiểm soát), không được để toàn bộ traffic đứng lại chỉ vì mất kết nối tới control plane.

```mermaid
sequenceDiagram
    participant Sidecar as Sidecar
    participant ControlPlane as Control Plane
    participant TargetPod as Target Pod đang cache

    Sidecar->>ControlPlane: Định kỳ đồng bộ danh sách endpoint
    ControlPlane-->>Sidecar: Danh sách endpoint hiện tại
    Sidecar->>Sidecar: Lưu ENDPOINT_CACHE_SNAPSHOT (cached_at, source=control_plane)

    Sidecar->>ControlPlane: Đồng bộ lần tiếp theo
    ControlPlane--xSidecar: Không phản hồi (control plane outage)

    Sidecar->>Sidecar: Phát hiện mất kết nối tới control plane
    Note over Sidecar: Không chặn traffic, tiếp tục dùng ENDPOINT_CACHE_SNAPSHOT gần nhất
    Sidecar->>TargetPod: Tiếp tục route request dựa trên cache
    TargetPod-->>Sidecar: Response OK
    Sidecar->>Sidecar: Vẫn tự health check các pod trong cache song song (fail open có kiểm soát)

    ControlPlane->>Sidecar: Control plane phục hồi
    Sidecar->>ControlPlane: Đồng bộ lại danh sách endpoint mới nhất
    Sidecar->>Sidecar: Cập nhật ENDPOINT_CACHE_SNAPSHOT, thay thế cache cũ
```
