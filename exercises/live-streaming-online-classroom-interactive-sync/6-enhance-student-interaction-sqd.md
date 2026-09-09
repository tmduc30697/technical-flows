# Enhance sequence — Gắn tương tác đúng mốc nội dung bài giảng

Đây là **enhance** của flow `student-interaction` đã có ở base. So với base (gắn `client_sent_at` là giờ đồng hồ client, không phản ánh nội dung đang giảng), flow này thay đổi ở chỗ: mỗi sự kiện tương tác được gắn thêm `lecture_timestamp_ms` — tính bằng vị trí nội dung bài giảng thực tại thời điểm học sinh tương tác, sau khi bù trừ độ trễ phân phối riêng của học sinh đó — đáp ứng yêu cầu 2.

```mermaid
sequenceDiagram
    actor Teacher
    participant Session as Class Session Store
    participant CDN
    actor Student as Hoc sinh (xem tre 5 giay)
    participant InteractionSvc as Interaction Service

    loop Lien tuc trong luc giang
        Teacher->>Session: Cap nhat lecture_position_ms theo noi dung dang giang
    end

    Teacher->>CDN: Dang giang noi dung tai lecture_position_ms = 1800000 (30:00)
    CDN-->>Student: Hoc sinh nhan noi dung nay tre 5 giay (dang xem vi tri 1795000)

    Student->>InteractionSvc: Dat cau hoi, client gui kem do tre playback do client tu do (vd 5000ms)
    InteractionSvc->>Session: Doc lecture_position_ms tai thoi diem client gui, tru di do tre playback
    Session-->>InteractionSvc: lecture_timestamp_ms = 1795000 (dung vi tri hoc sinh dang xem)
    InteractionSvc->>InteractionSvc: Luu INTERACTION_EVENT voi lecture_timestamp_ms = 1795000

    Teacher->>InteractionSvc: Xem lai danh sach cau hoi dang cho tra loi
    InteractionSvc-->>Teacher: Hien cau hoi kem lecture_timestamp_ms, giup giao vien biet dung no ung voi doan nao<br/>du hien tai giao vien da giang qua doan do
```
