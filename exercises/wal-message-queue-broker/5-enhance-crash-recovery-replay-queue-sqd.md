# Sequence Diagram — Enhance: Crash Recovery Replay Queue

Đây là **enhance**, flow mới hoàn toàn: sau crash, broker replay WAL để tái tạo đúng thứ tự hàng đợi và trạng thái ack, redeliver message đang xử lý dở khi crash cho consumer (có thể là consumer khác), đòi hỏi xử lý ở consumer phải idempotent.

```mermaid
sequenceDiagram
    participant Broker as Broker (restarted)
    participant WAL as WAL (disk)
    actor Consumer as Consumer (có thể khác consumer trước)

    Broker->>Broker: Khởi động lại sau crash
    Broker->>WAL: Đọc WAL_ENTRY theo thứ tự lsn

    loop mỗi MESSAGE, xét theo entry cuối cùng ghi nhận
        alt entry cuối là received hoặc delivered, chưa có acked
            Broker->>Broker: Coi message là chưa ack, đưa lại vào hàng đợi để giao tiếp
        else entry cuối là acked
            Broker->>Broker: Coi message đã xử lý xong, không đưa lại cho consumer đọc lần nữa
        end
    end

    Broker-->>Broker: Hàng đợi tái tạo đúng thứ tự và trạng thái ack tại thời điểm crash

    Broker->>Consumer: Giao lại các message chưa ack (kể cả message đã đọc dở trước crash)
    Consumer->>Consumer: Xử lý message (idempotent, chịu được đọc trùng)
    Consumer->>Broker: Ack sau khi xử lý xong
```
