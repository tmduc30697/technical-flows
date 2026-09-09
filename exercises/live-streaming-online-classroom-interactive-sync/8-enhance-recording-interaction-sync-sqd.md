# Enhance sequence — Đồng bộ marker tương tác trong bản ghi xem lại

Đây là **enhance**, flow hoàn toàn mới so với base (base không lưu marker đồng bộ giữa video và tương tác). Đáp ứng yêu cầu 4 trong đề bài: bản ghi lại buổi học phải giữ đồng bộ chính xác giữa video bài giảng và các mốc tương tác quan trọng (câu hỏi được trả lời, chuyển phần bài mới), không bị lệch do độ trễ tích luỹ qua nhiều giờ ghi hình.

```mermaid
sequenceDiagram
    actor Teacher
    participant InteractionSvc as Interaction Service
    participant Recorder as Recording Service
    participant RecordingStore as Recording + Marker Store
    actor StudentReview as Hoc sinh xem lai sau

    Teacher->>InteractionSvc: Danh dau da tra loi 1 cau hoi cu the (interaction_event_id)
    InteractionSvc->>Recorder: Bao su kien "question_answered" kem lecture_timestamp_ms goc cua cau hoi

    Note over Recorder: Recorder duy tri anh xa lien tuc giua lecture_timestamp_ms va recording_timestamp_ms thuc te,<br/>tu dieu chinh bu tru trong toan bo qua trinh ghi hinh nhieu gio

    Recorder->>Recorder: Quy doi lecture_timestamp_ms sang recording_timestamp_ms tuong ung tai thoi diem hien tai
    Recorder->>RecordingStore: Tao RECORDING_MARKER (marker_type=question_answered, recording_timestamp_ms da quy doi)

    Teacher->>InteractionSvc: Chuyen sang phan bai moi
    InteractionSvc->>Recorder: Bao su kien "new_topic"
    Recorder->>RecordingStore: Tao RECORDING_MARKER (marker_type=new_topic, recording_timestamp_ms hien tai)

    StudentReview->>RecordingStore: Xem lai buoi hoc, mo danh sach marker
    RecordingStore-->>StudentReview: Tra ve marker kem recording_timestamp_ms chinh xac
    StudentReview->>RecordingStore: Bam vao marker "cau hoi ve Dao ham"
    RecordingStore-->>StudentReview: Tua video dung ngay doan giao vien tra loi, khong bi lech du la gio thu 3 cua buoi ghi hinh
```
