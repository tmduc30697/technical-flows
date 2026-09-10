# Sequence Diagram — Enhance: Book Combo (Happy Path)

Đây là **enhance**, flow mới hoàn toàn gộp 3 lượt đặt riêng lẻ ([2-base-book-single-item-sqd.md](2-base-book-single-item-sqd.md)) thành một saga cho `COMBO_BOOKING`. Flow thể hiện yêu cầu thứ tự đặt phần dễ hủy/ít rủi ro trước (giữ chỗ tạm trước khi charge tiền thật), timeout riêng cho từng bước theo SLA của từng nhà cung cấp, và trạng thái real-time cho khách theo dõi.

```mermaid
sequenceDiagram
    actor Customer
    participant Orchestrator as Saga Orchestrator
    participant Hotel as Hotel Provider (Partner)
    participant Car as Car Rental Provider (Partner)
    participant Airline as Airline Provider (Partner)

    Customer->>Orchestrator: Book combo (flight, hotel, car)
    Orchestrator->>Orchestrator: Create COMBO_BOOKING, create SAGA_INSTANCE

    Orchestrator->>Hotel: Step 1, hold hotel room (low-risk, easy to release), timeout=10s
    Hotel-->>Orchestrator: Hotel held
    Orchestrator->>Orchestrator: Create SAGA_STEP (hotel_held), update combo status=in_progress
    Orchestrator-->>Customer: Real-time update, hotel secured

    Orchestrator->>Car: Step 2, hold rental car, timeout=10s
    Car-->>Orchestrator: Car held
    Orchestrator->>Orchestrator: Create SAGA_STEP (car_held)
    Orchestrator-->>Customer: Real-time update, car secured

    Orchestrator->>Airline: Step 3, charge and issue flight ticket (highest risk, done last), timeout=20s
    Airline-->>Orchestrator: Ticket issued, payment charged
    Orchestrator->>Orchestrator: Create SAGA_STEP (flight_booked, status=completed)

    Orchestrator->>Orchestrator: Mark SAGA_INSTANCE status=completed, COMBO_BOOKING status=confirmed
    Orchestrator-->>Customer: Combo booking fully confirmed
```
