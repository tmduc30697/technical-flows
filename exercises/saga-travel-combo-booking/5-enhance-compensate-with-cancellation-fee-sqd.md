# Sequence Diagram — Enhance: Compensate With Cancellation Fee

Đây là **enhance**, flow mới xử lý trường hợp bước charge vé máy bay thất bại/timeout sau khi hotel và car đã giữ chỗ thành công. Flow này thể hiện yêu cầu compensate phải log và tính đúng phí hủy (cancellation fee) khi hủy khách sạn/xe, không hoàn nhầm 100% khi có phí hủy.

```mermaid
sequenceDiagram
    participant Orchestrator as Saga Orchestrator
    participant Airline as Airline Provider (Partner)
    participant Hotel as Hotel Provider (Partner)
    participant Car as Car Rental Provider (Partner)
    actor Customer

    Orchestrator->>Airline: Step 3, charge and issue flight ticket
    Airline-->>Orchestrator: Request timed out, treated as failure

    Orchestrator->>Orchestrator: Create SAGA_STEP (flight_booked, status=failed)
    Orchestrator->>Orchestrator: Begin compensating completed steps in reverse order

    Orchestrator->>Car: Compensate, cancel car hold
    Car-->>Orchestrator: Cancelled, cancellation_fee applies (car was already confirmed, not just held)
    Orchestrator->>Orchestrator: Create CANCELLATION_FEE (step_id=car_held, fee_amount, refunded_amount=price-fee)

    Orchestrator->>Hotel: Compensate, cancel hotel hold
    Hotel-->>Orchestrator: Cancelled, no fee since it was only a soft hold
    Orchestrator->>Orchestrator: Create CANCELLATION_FEE (step_id=hotel_held, fee_amount=0, refunded_amount=full)

    Orchestrator->>Orchestrator: Mark SAGA_INSTANCE status=compensated, COMBO_BOOKING status=cancelled
    Orchestrator-->>Customer: Combo booking cancelled, refund minus car cancellation fee
```
