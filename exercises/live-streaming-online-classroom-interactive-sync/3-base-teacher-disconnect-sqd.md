# Base sequence — Xử lý giáo viên mất kết nối (naive)

Đây là **base**, mô tả cách hệ thống hiện tại xử lý khi giáo viên mất kết nối giữa buổi giảng: phiên học kết thúc ngay lập tức, học sinh kết nối lại phải vào 1 phiên mới, các tương tác phát sinh trong lúc gián đoạn bị thất lạc do không còn gắn với phiên nào còn hoạt động. Đây là tiền đề cho enhance yêu cầu 3 — khôi phục đúng vị trí trong buổi học mà không tạo phiên mới và không mất tương tác.

```mermaid
sequenceDiagram
    actor Teacher
    participant Ingest as Ingest Server
    participant Session as Class Session Store
    participant InteractionSvc as Interaction Service
    actor Student

    Teacher--x Ingest: Mat ket noi dot ngot giua buoi giang
    Ingest->>Session: UPDATE CLASS_SESSION SET status = ended, ended_at = now

    Student->>InteractionSvc: Van tiep tuc gui cau hoi trong luc gian doan
    InteractionSvc-->>Student: Loi, khong con CLASS_SESSION nao dang live de gan vao

    Teacher->>Ingest: Ket noi lai sau vai phut
    Ingest->>Session: Tao CLASS_SESSION moi (session_id khac)

    Note over Session,InteractionSvc: Cac cau hoi hoc sinh gui trong luc gian doan bi mat,<br/>khong duoc gan vao CLASS_SESSION moi vi da qua thoi diem session cu ket thuc
```
