# Enhance sequence — Hành vi producer/consumer trong lúc đang bầu leader mới

Đây là **enhance**, flow hoàn toàn mới phát sinh từ enhance — chưa tồn tại ở base vì base không có giai đoạn gián đoạn do bầu lại leader. Trong lúc chưa có leader mới, producer bị từ chối rõ ràng để tự retry (không buffer ngầm phía broker), còn consumer tạm dừng đúng tại offset cuối đã đọc, không đọc nhảy cóc hay đọc trùng khi kết nối lại broker mới — đáp ứng đúng yêu cầu 3 của đề bài. Cụm cũng chạy chaos test định kỳ mô phỏng đúng kịch bản này dưới tải cao (yêu cầu đo lường).

```mermaid
sequenceDiagram
    actor Producer
    actor Consumer
    participant Group as Partition Replica Group (đang bầu leader, chưa có leader)
    participant NewLeader as Broker Leader mới (sau khi bầu xong)

    par Trong lúc gián đoạn bầu leader
        Producer->>Group: Publish message
        Group-->>Producer: Reject rõ ràng "NO_LEADER, retry sau"
        Producer->>Producer: Tự backoff và retry lại sau, không mất message vì chưa từng được ack
    and
        Consumer->>Group: Fetch message tiếp theo từ offset=1000 (offset cuối đã đọc)
        Group-->>Consumer: Reject rõ ràng "NO_LEADER, tạm dừng"
        Consumer->>Consumer: Tạm dừng đúng tại offset=1000, không tự đoán/nhảy cóc
    end

    Note over Group: Election hoàn tất, NewLeader lên thay với last_committed_offset đã xác định đúng

    Producer->>NewLeader: Retry publish
    NewLeader-->>Producer: Ack "publish thành công" (sau khi commit majority)

    Consumer->>NewLeader: Fetch tiếp từ offset=1000
    NewLeader-->>Consumer: Trả đúng message kế tiếp, không lặp lại message đã đọc trước khi gián đoạn

    Note over Producer,Consumer: Chaos test định kỳ giết leader giữa lúc publish tải cao để xác nhận producer/consumer luôn theo đúng 2 hành vi này, không có message mất hoặc đảo thứ tự ngoài dự kiến
```
