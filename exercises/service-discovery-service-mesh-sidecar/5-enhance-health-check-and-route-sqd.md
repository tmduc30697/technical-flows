# Sequence Diagram — Enhance: Health Check And Route

Đây là **enhance**, flow health check thay đổi hẳn so với base: tách rõ liveness check (fail thì restart pod) và readiness check (fail thì chỉ loại khỏi routing tạm thời), có backoff khi liên tục fail để không tạo tải thêm lên service đang yếu, và ghi rõ `failure_reason` (timeout, HTTP 5xx, connection refused) thay vì chỉ trả unhealthy chung chung.

```mermaid
sequenceDiagram
    participant Sidecar as Sidecar
    participant TargetPod as Target Pod
    participant K8s as K8s Control Loop

    loop định kỳ, có backoff khi liên tục fail
        Sidecar->>TargetPod: Liveness check
        alt liveness fail
            TargetPod--xSidecar: Timeout/Connection refused
            Sidecar->>Sidecar: Ghi HEALTH_CHECK_RESULT (check_type=liveness, failure_reason)
            Sidecar->>K8s: Báo liveness fail, yêu cầu restart pod
        else liveness pass
            Sidecar->>TargetPod: Readiness check
            alt readiness fail
                TargetPod--xSidecar: HTTP 5xx
                Sidecar->>Sidecar: Ghi HEALTH_CHECK_RESULT (check_type=readiness, failure_reason=http_5xx)
                Sidecar->>Sidecar: Loại pod khỏi routing tạm thời, không restart
                Sidecar->>Sidecar: Tăng backoff_level, giãn chu kỳ check tiếp theo
            else readiness pass
                Sidecar->>Sidecar: Giữ pod trong routing table, reset backoff_level
            end
        end
    end
```
