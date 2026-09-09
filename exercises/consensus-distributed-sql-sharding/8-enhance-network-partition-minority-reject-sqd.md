# Enhance sequence — Network partition chia cluster, phía minority từ chối ghi

Đây là **enhance**, flow hoàn toàn mới phát sinh từ enhance — ở base cluster nhỏ và giả định không tách nhóm khi partition (không được đề cập). Enhance yêu cầu xử lý rõ kịch bản network partition chia cluster của 1 Raft group thành 2 nửa (ví dụ 2-3 hoặc 3-2), phía không đủ quorum tuyệt đối không được nhận ghi — đáp ứng đúng yêu cầu 5 của đề bài.

```mermaid
sequenceDiagram
    actor Client
    participant Minority as Node A, B (2 node, phía minority sau partition)
    participant Majority as Node C, D, E (3 node, phía majority, đã bầu leader mới)

    Note over Minority,Majority: Network partition chia Raft group của Range B thành 2 nửa 2-3

    Majority->>Majority: 3 node liên lạc được với nhau, đủ quorum (3/5)
    Majority->>Majority: Bầu/giữ leader hợp lệ ở phía majority, quorum_side = majority

    Client->>Minority: Write row(key=42) (client gửi nhầm/đang cache leader cũ ở phía minority)
    Minority->>Minority: Kiểm tra, chỉ liên lạc được 2/5 node, không đạt quorum 3/5
    Minority->>Minority: Đánh dấu quorum_side = minority, chuyển read-only/reject-write
    Minority-->>Client: Reject, "no quorum, cannot accept write"

    Client->>Majority: Retry write row(key=42) tới leader phía majority
    Majority->>Majority: Append + replicate tới 3/3 node còn liên lạc được, đạt majority của cả group (3/5)
    Majority-->>Client: 200 OK

    Note over Minority,Majority: Chỉ đúng 1 phía (majority) được phép ghi tại một thời điểm, tránh 2 phía cùng nhận ghi gây phân nhánh dữ liệu (split-brain)
```
