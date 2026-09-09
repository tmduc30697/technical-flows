# Enhance sequence — Guardian recovery (người thân/giám hộ hợp pháp)

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đáp ứng yêu cầu thứ 5 của đề bài: xử lý trường hợp bệnh nhân không tự thao tác được (bất tỉnh, trẻ em, người cao tuổi) — quy trình này xác minh mối quan hệ hợp pháp một cách có kiểm soát và **tách biệt hẳn** khỏi flow "Password reset" tự khôi phục của chính chủ tài khoản.

```mermaid
sequenceDiagram
    actor Guardian as Người thân/Giám hộ
    participant App as Healthcare Platform
    participant DB as GUARDIAN_RELATIONSHIP store
    participant Verify as Legal Verification Team
    participant Request as GUARDIAN_RECOVERY_REQUEST store

    Guardian->>App: Yêu cầu khôi phục quyền truy cập thay cho bệnh nhân (không dùng flow tự khôi phục)
    Guardian->>App: Nộp bằng chứng quan hệ hợp pháp (giấy khai sinh, giấy ủy quyền giám hộ, giấy tờ tùy thân của guardian)
    App->>DB: Tạo GUARDIAN_RELATIONSHIP (relationship_type, legal_proof_ref, verification_status=pending)
    DB->>Verify: Chuyển hồ sơ cho đội xác minh pháp lý
    Verify->>Verify: Đối chiếu độc lập giấy tờ + xác minh với bệnh nhân/cơ sở dữ liệu hộ tịch nếu cần
    alt Quan hệ hợp lệ
        Verify->>DB: verification_status=verified
        App->>Request: Tạo GUARDIAN_RECOVERY_REQUEST (status=pending)
        Request->>Request: Xử lý khôi phục quyền truy cập được ủy quyền (vd cấp truy cập giới hạn thay mặt bệnh nhân)
        Request-->>Guardian: Cấp quyền truy cập được ủy quyền, gắn với GUARDIAN_RELATIONSHIP đã verify
    else Quan hệ không xác minh được
        Verify->>DB: verification_status=rejected
        DB-->>Guardian: Từ chối, không cấp quyền truy cập nào
    end
```
