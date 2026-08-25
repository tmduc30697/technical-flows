# Technical Flows

Danh sách tập trung vào chiều sâu kỹ thuật/hạ tầng trong bối cảnh web (đã bỏ các flow business logic/CRUD thuần ít giá trị kỹ thuật, và loại các flow gắn phần cứng/IoT không liên quan), sắp theo độ nặng kỹ thuật tăng dần:

## Xác thực/bảo mật chuẩn hóa, nhiều web app đều cần

- [OAuth authorization code flow](01-auth-security/oauth-authorization-code-flow.md)
- [Single Sign-On (SSO) flow](01-auth-security/sso-flow.md)
- [Multi-factor authentication (MFA) flow](01-auth-security/mfa-flow.md)
- [Password reset & Account recovery flow](01-auth-security/password-reset-account-recovery-flow.md)
- [Device trust/session flow](01-auth-security/device-trust-session-flow.md)

## Kỹ thuật rõ ràng nhưng gói gọn trong phạm vi một service/module

- [Load balancing algorithms flow (round-robin/least-connection/consistent hashing ở layer LB)](02-service-module/load-balancing-algorithms-flow.md)
- [Service discovery & health check flow](02-service-module/service-discovery-health-check-flow.md)
- [Graceful shutdown & connection draining flow](02-service-module/graceful-shutdown-connection-draining-flow.md)
- [Distributed tracing/observability pipeline flow](02-service-module/distributed-tracing-observability-pipeline-flow.md)
- [Blue-green/Canary deployment flow](02-service-module/blue-green-canary-deployment-flow.md)
- [Multi-tenancy/tenant data isolation flow](02-service-module/multi-tenancy-tenant-data-isolation-flow.md)

## Xử lý dữ liệu lớn/pipeline, đặc trưng cho hệ thống web quy mô lớn

- [Video upload & transcoding pipeline flow](03-data-pipeline/video-upload-transcoding-pipeline-flow.md)
- [Live streaming ingestion flow](03-data-pipeline/live-streaming-ingestion-flow.md)
- [Reconciliation flow giữa nhiều hệ thống thanh toán](03-data-pipeline/reconciliation-flow.md)
- [Search indexing/autocomplete pipeline flow](03-data-pipeline/search-indexing-autocomplete-pipeline-flow.md)

## Race condition, xử lý đồng thời cao, timing-sensitive

- [Deadlock detection & transaction isolation levels flow](04-race-condition/deadlock-detection-transaction-isolation-levels-flow.md)
- [Zero-downtime schema migration & dual-write flow](04-race-condition/zero-downtime-schema-migration-dual-write-flow.md)
- [Flash sale/high-concurrency checkout flow](04-race-condition/flash-sale-high-concurrency-checkout-flow.md)
- [Inventory reservation flow](04-race-condition/inventory-reservation-flow.md)
- [Payment flow (xử lý giao dịch + callback + đối soát)](04-race-condition/payment-flow.md)
- [Optimistic vs pessimistic locking flow](04-race-condition/optimistic-vs-pessimistic-locking-flow.md)

## Concurrency, real-time, phân tán ở mức lõi backend web

- [Consensus algorithm flow (Raft/Paxos: leader election, log replication)](05-distributed-systems/consensus-algorithm-flow.md)
- [Distributed lock & leader election flow (lease-based, Redis/ZooKeeper)](05-distributed-systems/distributed-lock-leader-election-flow.md)
- [Event sourcing/Saga pattern flow (rollback/compensate xuyên nhiều service)](05-distributed-systems/event-sourcing-saga-pattern-flow.md)
- [Consistent hashing & sharding/partitioning flow](05-distributed-systems/consistent-hashing-sharding-partitioning-flow.md)
- [CAP theorem & tunable consistency (quorum read/write) flow](05-distributed-systems/cap-theorem-tunable-consistency-flow.md)
- [Write-ahead log (WAL) & crash recovery/durability flow](05-distributed-systems/write-ahead-log-wal-crash-recovery-durability-flow.md)
- [Circuit breaker & retry with backoff flow](05-distributed-systems/circuit-breaker-retry-with-backoff-flow.md)
- [Cache invalidation flow](05-distributed-systems/cache-invalidation-flow.md)
- [Rate limiting/throttling flow](05-distributed-systems/rate-limiting-throttling-flow.md)
