# Enhance sequence — Lost-channels recovery (xác minh danh tính bán tự động)

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (khác với flow "Channel-based recovery" ở base, vốn chỉ hoạt động khi còn giữ 1 kênh). Đáp ứng yêu cầu 1, 2 và 5 của đề bài: chuyển sang xác minh danh tính thủ công/bán tự động, đóng băng rút/chuyển tiền nhưng vẫn cho xem thông tin, và yêu cầu **nhiều bằng chứng độc lập** để chống social engineering nhắm vào bộ phận hỗ trợ.

```mermaid
sequenceDiagram
    actor User
    participant App as Fintech App
    participant DB as USER / RECOVERY_CASE store
    participant FaceMatch as Face-match Service
    actor Agent as Support Agent
    participant Audit as AUDIT_LOG

    User->>App: Yêu cầu khôi phục, không còn email/phone nào khớp bản ghi
    App->>DB: Tạo RECOVERY_CASE (reason=lost_all_channels, status=pending_verification)
    App->>DB: Tạo ACCOUNT_RESTRICTION (type=withdrawal_frozen + view_only)
    DB->>Audit: Ghi log tạo case (immutable)
    App-->>User: Thông báo tài khoản tạm đóng băng rút/chuyển tiền, vẫn xem được số dư/lịch sử
    User->>App: Upload giấy tờ tùy thân (IDENTITY_EVIDENCE type=id_document)
    User->>App: Chụp ảnh khớp khuôn mặt (IDENTITY_EVIDENCE type=selfie_face_match)
    App->>FaceMatch: Đối chiếu ảnh selfie với ảnh trên giấy tờ
    FaceMatch-->>App: Kết quả match_status
    App->>DB: Lưu cả 2 IDENTITY_EVIDENCE kèm kết quả
    DB->>Audit: Ghi log từng bằng chứng nộp (immutable)
    App->>Agent: Đưa case vào hàng chờ review (kèm đủ 2 bằng chứng độc lập, không hỏi 1 câu bảo mật duy nhất)
    Agent->>Agent: Đối chiếu độc lập giấy tờ + face-match + lịch sử tài khoản
    Agent->>DB: Ghi SUPPORT_REVIEW (decision=approved/rejected, evidence_considered)
    DB->>Audit: Ghi log quyết định (ai duyệt, dựa trên bằng chứng gì, khi nào — immutable)
    alt Approved
        DB->>DB: RECOVERY_CASE status=approved
        DB-->>User: Case tiếp tục sang flow "Post-recovery session reset & cooling-off"
    else Rejected
        DB->>DB: RECOVERY_CASE status=rejected
        DB-->>User: Thông báo từ chối, giữ nguyên ACCOUNT_RESTRICTION
    end
```
