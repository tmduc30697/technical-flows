# Sequence Diagram — Enhance: Handle Network Partition

Đây là **enhance**, flow hoàn toàn mới mô phỏng network partition giữa hai registry — yêu cầu đề bài rằng hệ thống không được tự ý coi region kia là chết hoàn toàn chỉ vì mất liên lạc giữa hai registry, mà phải phân biệt "mất đồng bộ registry" với "instance thật sự unhealthy".

```mermaid
sequenceDiagram
    participant RegistryA as Registry (Region A)
    participant RegistryB as Registry (Region B)
    actor Ops as Ops Team

    RegistryA->>RegistryB: Periodic sync heartbeat
    RegistryB--xRegistryA: No response (simulated network partition)
    RegistryA->>RegistryA: Update REGISTRY_SYNC_STATE, partition_status=suspected

    loop retry with backoff
        RegistryA->>RegistryB: Retry sync
        RegistryB--xRegistryA: Still unreachable
    end

    RegistryA->>RegistryA: After grace period, mark partition_status=confirmed
    Note over RegistryA: Không tự coi toàn bộ instance của Region B là chết
    Note over RegistryA: Chỉ ngưng tin dữ liệu mới từ Region B, vẫn giữ danh sách cache gần nhất để route khi cần
    RegistryA->>Ops: Alert network partition detected between regions

    RegistryB->>RegistryA: Sync recovers, heartbeat resumes
    RegistryA->>RegistryA: Update partition_status=resolved, resume trusting Region B data
```
