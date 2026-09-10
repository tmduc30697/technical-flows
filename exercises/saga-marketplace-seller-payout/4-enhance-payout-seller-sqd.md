# Sequence Diagram — Enhance: Payout Seller

Đây là **enhance**, flow "chi trả cho seller" đã có ở base ([2-base-payout-seller-sqd.md](2-base-payout-seller-sqd.md)) nhưng nay thay đổi căn bản: mỗi seller có một `SAGA_INSTANCE` riêng, mỗi bước ghi `SAGA_STEP`/`SAGA_EVENT`, và lệnh chuyển tiền dùng `idempotency_key` cố định để retry an toàn thay vì gọi trực tiếp không kiểm soát như base.

```mermaid
sequenceDiagram
    participant Orchestrator as Saga Orchestrator
    participant Balance as Seller Balance Service
    participant Partner as Payment Partner

    Orchestrator->>Orchestrator: Create SAGA_INSTANCE for this seller's PAYOUT

    Orchestrator->>Balance: Step 1, deduct total_amount from SELLER_BALANCE
    Balance-->>Orchestrator: Balance deducted
    Orchestrator->>Orchestrator: Create SAGA_STEP (deduct_balance, status=completed), log event

    Orchestrator->>Partner: Step 2, initiate transfer (amount, idempotency_key=payout_id)
    Partner-->>Orchestrator: Transfer succeeded

    Orchestrator->>Orchestrator: Create SAGA_STEP (initiate_transfer, status=completed), log event
    Orchestrator->>Orchestrator: Mark SAGA_INSTANCE status=completed, PAYOUT status=paid
```
