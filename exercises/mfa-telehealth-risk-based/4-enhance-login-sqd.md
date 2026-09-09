# Enhance sequence — Login với adaptive MFA theo rủi ro

Đây là **enhance**, cùng flow "Đăng nhập" đã có ở base nhưng nay thay đổi hoàn toàn: sau khi đúng mật khẩu, hệ thống tính điểm rủi ro (thiết bị, vị trí, giờ truy cập) rồi mới quyết định có yêu cầu OTP hay không, thay vì luôn hỏi OTP như base. Bác sĩ vẫn luôn bị yêu cầu MFA mỗi phiên bất kể rủi ro (giữ nguyên hành vi base cho vai trò này), còn bệnh nhân chỉ bị hỏi khi thiết bị/vị trí chưa từng ghi nhận. Nếu dịch vụ chấm điểm rủi ro lỗi/timeout, hệ thống fail-safe về yêu cầu MFA đầy đủ. Mọi quyết định đều được log lại lý do.

```mermaid
sequenceDiagram
    actor User as Người dùng (bệnh nhân hoặc bác sĩ)
    participant App as Auth Service
    participant Risk as Risk Scoring Service
    participant DB as LOGIN_SESSION / KNOWN_DEVICE store
    participant Log as MFA_DECISION_LOG

    User->>App: Nhập email + mật khẩu (kèm device fingerprint, IP)
    App->>App: Kiểm tra password_hash
    App->>DB: Tạo LOGIN_SESSION(device_id, ip_address, geo_country, geo_city)

    App->>Risk: Yêu cầu chấm điểm rủi ro (device_known, location_known, time_anomaly)
    alt Risk service phản hồi bình thường
        Risk-->>App: RISK_ASSESSMENT(risk_score, risk_level)
        alt role = doctor
            App->>Log: Ghi quyết định mfa_required=true, reason="doctor_role"
            Note over App: Bác sĩ luôn bắt buộc MFA mỗi phiên, không phân biệt rủi ro
        else role = patient và risk_level = low (thiết bị/vị trí đã biết)
            App->>Log: Ghi quyết định mfa_required=false, reason="known_device_location"
            App->>DB: Cập nhật LOGIN_SESSION(mfa_verified=true, mfa_required=false)
            App-->>User: Đăng nhập thành công, không cần OTP
        else role = patient và risk_level = high (thiết bị/vị trí chưa từng ghi nhận)
            App->>Log: Ghi quyết định mfa_required=true, reason="unknown_device_or_location"
        end
    else Risk service lỗi hoặc timeout
        Risk-->>App: Lỗi/timeout
        App->>Log: Ghi quyết định mfa_required=true, reason="risk_service_timeout_fail_safe"
        Note over App: Fail-safe, không bao giờ mặc định bỏ qua MFA vì lỗi hạ tầng
    end

    opt mfa_required = true (mọi nhánh trừ patient risk_level=low)
        App->>DB: Sinh OTP_CODE cho session
        App-->>User: Yêu cầu nhập OTP
        User->>App: Nhập OTP
        App->>DB: Kiểm tra OTP_CODE hợp lệ và chưa dùng
        App->>DB: Cập nhật LOGIN_SESSION(mfa_verified=true)
        App->>DB: Cập nhật/tạo KNOWN_DEVICE cho user này
        App-->>User: Đăng nhập thành công, cho truy cập hồ sơ bệnh án
    end
```
