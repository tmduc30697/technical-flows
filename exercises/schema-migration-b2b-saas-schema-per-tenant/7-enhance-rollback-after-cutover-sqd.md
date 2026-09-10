# Sequence Diagram — Enhance: Rollback After Cutover

Đây là **enhance**, flow mới xử lý khi phát hiện lỗi dữ liệu ở schema mới sau cutover: trả routing của tenant đó về schema cũ trong vòng vài phút mà không mất giao dịch nào đã ghi vào schema mới trong khoảng thời gian đã cutover. Flow này là checklist rollback được đề bài yêu cầu tường minh.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Migration as Migration Job Service
    participant Routing as Tenant Routing Table
    participant NewDB as New Schema
    participant OldDB as Old Schema
    participant App as HR App Service

    Ops->>Migration: Report data issue found in New Schema for tenant_id, request rollback

    Migration->>NewDB: Extract all writes made to New Schema since cutover_at
    NewDB-->>Migration: List of changes since cutover

    Migration->>OldDB: Replay/apply those changes onto Old Schema
    OldDB-->>Migration: Old Schema now caught up with post-cutover changes

    Migration->>Routing: Update TENANT_ROUTING.schema_state = old for tenant_id
    Routing-->>Migration: Routing reverted

    Routing-->>App: Subsequent writes for this tenant now route back to Old Schema
    Migration-->>Ops: Rollback completed, no post-cutover transaction lost

    Note over Migration: New Schema is left as-is for investigation
    Note over Migration: not deleted, so the root cause of the data issue can still be diagnosed
```
