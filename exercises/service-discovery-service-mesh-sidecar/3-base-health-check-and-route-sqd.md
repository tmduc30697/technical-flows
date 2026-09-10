# Sequence Diagram — Base: Health Check And Route

Đây là **base**, flow sidecar health check các endpoint đang giao tiếp bằng một loại kiểm tra chung chung rồi loại pod lỗi khỏi routing — chưa phân biệt liveness/readiness, chưa có backoff, chưa báo rõ nguyên nhân lỗi. Đây chính là điểm sẽ thay đổi ở enhance.

```mermaid
sequenceDiagram
    participant Sidecar as Sidecar
    participant TargetPod as Target Pod

    loop định kỳ
        Sidecar->>TargetPod: Health check
        alt pass
            TargetPod-->>Sidecar: OK
            Sidecar->>Sidecar: Giữ pod trong routing table
        else fail
            TargetPod--xSidecar: Timeout/lỗi
            Sidecar->>Sidecar: Đánh unhealthy, loại khỏi routing
        end
    end
```
