# Enhance sequence — Huấn luyện mô hình phân tích trên dữ liệu đã ẩn danh hóa

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base chưa có pipeline AI/phân tích dữ liệu tổng hợp nào. Đáp ứng yêu cầu 5 của đề bài: tính năng phân tích xu hướng bệnh chỉ được huấn luyện trên dữ liệu đã ẩn danh hóa đúng cách, và mô hình không được "nhớ" rồi lộ lại thông tin cụ thể của một bệnh nhân cho tenant khác truy vấn.

```mermaid
sequenceDiagram
    participant Job as Anonymization Job (định kỳ)
    participant DB as MEDICAL_RECORD (nhiều tenant, dedicated DB/schema)
    participant Anon as ANONYMIZED_PATIENT_RECORD store
    participant Trainer as Model Training Pipeline
    participant Model as ANALYTICS_MODEL
    actor Analyst as Nhân viên phân tích (tenant bất kỳ)

    Job->>DB: Đọc bản ghi bệnh nhân theo từng tenant (trong phạm vi cách ly của tenant đó)
    Job->>Job: Loại bỏ/generalize thông tin định danh trực tiếp và gián tiếp
    Job->>Anon: Ghi ANONYMIZED_PATIENT_RECORD(anonymization_method, tenant_id chỉ để trace nội bộ)
    Note over Job,Anon: Bước ẩn danh hóa chạy trong phạm vi cách ly của từng tenant, dữ liệu định danh gốc không rời khỏi ranh giới tenant

    Trainer->>Anon: Lấy tập dữ liệu đã ẩn danh hóa, gộp nhiều tenant
    Trainer->>Trainer: Huấn luyện/cập nhật mô hình thống kê xu hướng
    Trainer->>Model: Lưu ANALYTICS_MODEL(version, trained_at, training_scope=anonymized_only)

    Analyst->>Model: Truy vấn xu hướng bệnh (ví dụ theo khu vực, độ tuổi)
    Model-->>Analyst: Trả kết quả tổng hợp/thống kê (aggregate), không trả bản ghi cá nhân
    Note over Model: Mô hình chỉ trả aggregate, có kiểm tra k-anonymity trước khi trả kết quả để tránh suy luận ngược ra một bệnh nhân cụ thể của tenant khác
```
