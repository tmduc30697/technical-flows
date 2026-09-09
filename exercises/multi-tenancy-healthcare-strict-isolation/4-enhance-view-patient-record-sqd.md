# Enhance sequence — Xem hồ sơ bệnh nhân (database/schema riêng, khóa riêng theo tenant)

Đây là **enhance**, cùng flow "Xem hồ sơ bệnh nhân" đã có ở base nhưng nay thay đổi theo yêu cầu 1 và 2 của đề bài: dữ liệu nằm trong database/schema riêng của từng tenant (không còn chỉ lọc `tenant_id` trong shared schema), và giải mã bằng khóa mã hóa riêng của chính tenant đó thay vì khóa dùng chung.

```mermaid
sequenceDiagram
    actor Staff as Nhân viên phòng khám (tenant A)
    participant App as Clinic Records Service
    participant Router as Tenant Connection Router
    participant DB as Dedicated DB/schema của tenant A
    participant Key as TENANT_ENCRYPTION_KEY (riêng tenant A)
    participant Log as AUDIT_LOG (partition riêng tenant A)

    Staff->>App: Yêu cầu xem hồ sơ bệnh nhân X
    App->>Router: Xác định connection/schema đích theo tenant_id=A
    Router-->>App: Kết nối tới dedicated DB/schema của tenant A
    App->>DB: SELECT * FROM medical_record WHERE patient_id = X
    Note over DB: Không cần lọc tenant_id thủ công nữa vì đã cách ly ở tầng DB/schema, giảm rủi ro lỗi logic filter
    DB-->>App: Bản ghi đã mã hóa
    App->>Key: Lấy TENANT_ENCRYPTION_KEY của tenant A
    Key-->>App: key_material (chỉ dùng được cho dữ liệu tenant A)
    App->>App: Giải mã nội dung hồ sơ
    App->>Log: Ghi AUDIT_LOG vào partition riêng của tenant A
    App-->>Staff: Trả về nội dung hồ sơ
```
