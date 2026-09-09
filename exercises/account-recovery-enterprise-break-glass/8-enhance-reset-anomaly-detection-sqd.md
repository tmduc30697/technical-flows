# Enhance sequence — Reset anomaly detection

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đáp ứng yêu cầu thứ 5 của đề bài: phát hiện khi 1 helpdesk agent thực hiện reset cho số lượng tài khoản lớn bất thường trong thời gian ngắn — dấu hiệu agent bị mạo danh hoặc lạm quyền.

```mermaid
sequenceDiagram
    participant Audit as AUDIT_LOG
    participant Monitor as Anomaly Monitor
    participant DB as ANOMALY_ALERT store
    participant Security as Security/IT Lead

    loop Mỗi khi có bản ghi reset mới
        Audit-->>Monitor: Sự kiện reset mới (agent_id, timestamp)
        Monitor->>Monitor: Tính reset_count của agent trong rolling window
        alt reset_count vượt threshold
            Monitor->>DB: Tạo ANOMALY_ALERT (agent_id, window, reset_count, status=open)
            DB-->>Security: Cảnh báo agent nghi ngờ bị mạo danh/lạm quyền
            Security->>Security: Rà soát các RESET_TICKET gần đây của agent đó
            alt Xác nhận có lạm quyền/mạo danh
                Security->>DB: Khoá tạm quyền reset của agent, status=confirmed
            else Hoạt động hợp lệ (vd đợt onboarding lớn)
                Security->>DB: Đóng alert, status=false_positive
            end
        else Trong ngưỡng bình thường
            Monitor->>Monitor: Không tạo alert
        end
    end
```
