# Base sequence — Học sinh tương tác theo giờ đồng hồ client (naive)

Đây là **base**, mô tả flow học sinh giơ tay/đặt câu hỏi ở trạng thái hiện tại: sự kiện được gắn `client_sent_at` là giờ đồng hồ trên máy học sinh tại thời điểm bấm, không phản ánh vị trí thực trong bài giảng mà học sinh đang xem (do độ trễ phân phối). Đây là tiền đề cho enhance yêu cầu 2 — cần gắn đúng mốc nội dung bài giảng, không phải giờ bấm.

```mermaid
sequenceDiagram
    actor Teacher
    participant CDN
    actor Student as Hoc sinh (xem tre 5 giay so voi giao vien)
    participant InteractionSvc as Interaction Service

    Teacher->>CDN: Dang giang noi dung "Phan 3: Dao ham" luc T=00:30:00
    CDN-->>Student: Hoc sinh nhan duoc noi dung nay luc T=00:30:05 (tre 5 giay)

    Student->>InteractionSvc: Dat cau hoi ve noi dung dang xem, client_sent_at = 00:30:05
    InteractionSvc->>InteractionSvc: Luu INTERACTION_EVENT voi client_sent_at = 00:30:05

    Note over Teacher,InteractionSvc: Giao vien thuc te dang giang o T=00:30:10 khi nhan duoc cau hoi,<br/>nhung khong ro cau hoi nay ung voi noi dung luc T=00:30:05 hay T=00:30:10 cua chinh giao vien
```
