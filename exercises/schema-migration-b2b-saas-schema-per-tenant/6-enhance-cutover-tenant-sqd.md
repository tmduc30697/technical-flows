# Sequence Diagram — Enhance: Cutover Tenant

Đây là **enhance**, flow mới thực hiện cutover cho một tenant: khóa ghi tạm thời (vài giây) chỉ cho riêng tenant đó, chuyển routing từ old sang new schema, rồi mở khóa — trong khi các tenant khác không bị ảnh hưởng. Flow này thể hiện đúng tình huống "request đang giữa transaction ghi vào schema cũ ngay tại thời điểm chuyển routing".

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Migration as Migration Job Service
    participant Lock as Cutover Lock Service
    participant Routing as Tenant Routing Table
    participant OldDB as Old Schema
    participant NewDB as New Schema
    participant App as HR App Service

    Ops->>Migration: Trigger cutover for tenant_id (backfill already verified complete)
    Migration->>Lock: Acquire CUTOVER_LOCK for this tenant_id only

    Lock->>App: Reject/queue new write requests for this tenant_id
    Note over Lock,App: Other tenants' requests are completely unaffected

    Migration->>OldDB: Wait for any in-flight transaction for this tenant to finish
    OldDB-->>Migration: All in-flight writes drained

    Migration->>NewDB: Final incremental sync, catch up any last changes
    NewDB-->>Migration: Fully caught up

    Migration->>Routing: Update TENANT_ROUTING.schema_state = new for tenant_id
    Routing-->>Migration: Routing updated

    Migration->>Lock: Release CUTOVER_LOCK
    Lock->>App: Resume accepting writes, now routed to New Schema

    Migration-->>Ops: Cutover completed for tenant, maintenance window closed
```
