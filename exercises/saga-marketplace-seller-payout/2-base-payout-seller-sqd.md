# Sequence Diagram — Base: Payout Seller

Đây là **base**, flow "chi trả cho seller" gọi tuần tự: tính hoa hồng, trừ số dư, rồi gọi đối tác chuyển tiền — tiền đề cho saga. Ở base, flow này còn đơn giản, chưa có compensate an toàn: nếu bước gọi đối tác timeout hoặc lỗi, không rõ nên hoàn số dư hay không, dễ dẫn tới double-pay hoặc mất tiền oan cho seller.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Payout as Payout Service
    participant Balance as Seller Balance Service
    participant Partner as Payment Partner

    Ops->>Payout: Trigger payout run for eligible sellers
    Payout->>Payout: Aggregate completed ORDER into PAYOUT (total_amount)

    Payout->>Balance: Deduct total_amount from SELLER_BALANCE
    Balance-->>Payout: Balance deducted

    Payout->>Partner: Initiate transfer (amount, seller bank info)
    Partner-->>Payout: Transfer result (success/failure/timeout)

    alt transfer succeeds
        Payout->>Payout: Mark PAYOUT status=paid
    else transfer fails or times out
        Payout->>Payout: Unclear whether money actually moved
        Note over Payout,Partner: Naive compensate would just refund the balance here
        Note over Payout,Partner: risking double-pay if the transfer actually went through
    end

    Payout-->>Ops: Payout run summary
```
