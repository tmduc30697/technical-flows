# Enhance sequence — Khôi phục đúng vị trí buổi học khi giáo viên gián đoạn

Đây là **enhance** của flow `teacher-disconnect` đã có ở base. So với base (kết thúc phiên ngay, tạo phiên mới khi kết nối lại, mất tương tác trong lúc gián đoạn), flow này thay đổi ở chỗ: phiên chuyển sang trạng thái `interrupted` kèm `reconnect_deadline`, các `INTERACTION_EVENT` gửi trong lúc gián đoạn vẫn được nhận và gắn vào đúng `class_session_id` cũ, giáo viên kết nối lại được khôi phục đúng `lecture_position_ms` mà không tạo phiên mới — đáp ứng yêu cầu 3.

```mermaid
sequenceDiagram
    actor Teacher
    participant Ingest as Ingest Server
    participant Session as Class Session Store
    participant InteractionSvc as Interaction Service
    actor Student

    Teacher--x Ingest: Mat ket noi dot ngot giua buoi giang (lecture_position_ms = 1800000)
    Ingest->>Session: UPDATE status = interrupted, interrupted_at = now,<br/>reconnect_deadline = now + N phut
    Session-->>Student: Hien thi "giao vien tam gian doan", giu nguyen man hinh lop hoc

    Student->>InteractionSvc: Van gui cau hoi trong luc gian doan
    InteractionSvc->>Session: Gan INTERACTION_EVENT vao class_session_id hien tai (van con ton tai, chi la interrupted)
    Session-->>InteractionSvc: Luu thanh cong, khong bi mat

    alt Teacher ket noi lai truoc reconnect_deadline
        Teacher->>Ingest: Ket noi lai
        Ingest->>Session: UPDATE status = live, xoa interrupted_at/reconnect_deadline
        Session-->>Teacher: Khoi phuc dung lecture_position_ms = 1800000 truoc do
        Session->>InteractionSvc: Chuyen giao toan bo INTERACTION_EVENT phat sinh trong luc gian doan
        InteractionSvc-->>Teacher: Hien thi day du cac cau hoi/gio tay da phat sinh trong luc gian doan
    else Qua reconnect_deadline
        Ingest->>Session: UPDATE status = ended, ended_at = now
        Session-->>Student: Bao buoi hoc da thuc su ket thuc
    end
```
