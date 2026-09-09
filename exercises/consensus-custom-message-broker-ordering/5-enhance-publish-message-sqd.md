# Enhance sequence — Publish message (ack chỉ sau khi replicate majority)

Đây là **enhance** của flow đã có ở base (`2-base-publish-message-sqd.md`). So với base, offset chỉ được gán và message chỉ coi là "đã publish thành công" (ack cho producer) sau khi leader replicate thành công tới majority broker trong `PARTITION_REPLICA_GROUP`, không phải ngay khi leader nhận được — đáp ứng đúng yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    actor Producer
    participant Leader as Broker Leader (partition P1, term=4)
    participant F1 as Follower 1
    participant F2 as Follower 2 (3 broker/nhóm)

    Producer->>Leader: Publish message vào partition P1
    Leader->>Leader: Gán offset=1000 tạm thời, committed=false
    par Replicate tới follower
        Leader->>F1: Replicate(offset=1000, term=4)
        Leader->>F2: Replicate(offset=1000, term=4)
    end
    F1-->>Leader: ACK
    F2-->>Leader: ACK

    Leader->>Leader: Đếm ACK = leader + F1 + F2 = 3/3 (đạt majority của nhóm)
    Leader->>Leader: Đánh dấu MESSAGE(offset=1000, committed=true)
    Leader-->>Producer: Ack "publish thành công", offset=1000

    Note over Leader,F2: Nếu chỉ leader nhận được mà chưa replicate xong, offset=1000 vẫn ở trạng thái chưa commit và KHÔNG được ack cho producer
```
