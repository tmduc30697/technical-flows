# Sequence Diagram — Enhance: Request And Complete Trip (Happy Path)

Đây là **enhance**, flow vòng đời chuyến đi đã có ở base ([2-base-request-and-complete-trip-sqd.md](2-base-request-and-complete-trip-sqd.md)) nhưng nay thay đổi căn bản: một Saga Orchestrator điều phối từng bước xuyên Matching/Trip/Payment, mỗi bước được ghi `SAGA_STEP`/`SAGA_EVENT` để có thể resume và audit lại toàn bộ hành trình.

```mermaid
sequenceDiagram
    actor Rider
    actor Driver
    participant Orchestrator as Saga Orchestrator
    participant Matching as Matching Service
    participant Payment as Payment Service

    Rider->>Orchestrator: Request trip
    Orchestrator->>Orchestrator: Create TRIP, create SAGA_INSTANCE (current_step=matching)

    Orchestrator->>Matching: Find nearby driver
    Matching-->>Orchestrator: Driver found
    Orchestrator->>Orchestrator: Create SAGA_STEP (matched), log event
    Orchestrator-->>Driver: Trip offer
    Driver->>Orchestrator: Accept trip

    Driver->>Orchestrator: Mark picked up
    Orchestrator->>Orchestrator: Create SAGA_STEP (picked_up), log event

    Driver->>Orchestrator: Mark trip completed
    Orchestrator->>Orchestrator: Create SAGA_STEP (trip_completed), log event

    Orchestrator->>Payment: Charge rider for trip
    Payment-->>Orchestrator: Payment captured
    Orchestrator->>Orchestrator: Create SAGA_STEP (payment_settled, status=completed), log event

    Orchestrator->>Orchestrator: Mark SAGA_INSTANCE status=completed
    Orchestrator-->>Rider: Trip completed, receipt
```
