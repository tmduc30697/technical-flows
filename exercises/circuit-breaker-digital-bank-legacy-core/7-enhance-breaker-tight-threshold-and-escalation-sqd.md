# Enhance sequence — Breaker tight threshold & escalation

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có breaker riêng cho core banking). Đáp ứng yêu cầu 1 và 4 của đề bài: ngưỡng mở thấp hơn bình thường để ưu tiên bảo vệ core banking, và escalate khẩn ngay khi breaker mở kéo dài.

```mermaid
sequenceDiagram
    participant Breaker as CIRCUIT_BREAKER_STATE (core_banking)
    participant Core as Core Banking (legacy)
    participant Escalation as BREAKER_ESCALATION store
    actor Ops as Team vận hành

    Note over Breaker: error_rate_threshold được cấu hình THẤP HƠN bình thường — mở sớm hơn để bảo vệ core dễ sập
    loop Theo dõi liên tục
        Breaker->>Core: Ghi nhận tỉ lệ lỗi/latency
    end
    alt Vượt ngưỡng thấp đã cấu hình (ưu tiên bảo vệ core hơn giữ app luôn phản hồi)
        Breaker->>Breaker: Chuyển state=open, opened_at=now
        Note over Core: Core được nghỉ ngay, dù nghĩa là app phải fallback (xem flow "View balance") hoặc từ chối giao dịch ghi
    end

    loop Theo dõi thời gian breaker mở
        Escalation->>Breaker: Kiểm tra state và opened_at
        alt Vẫn đang open và đã vượt ngưỡng thời gian cho phép (core down lâu)
            Escalation->>Escalation: Ghi BREAKER_ESCALATION(duration_ms, escalation_channel=phone/pager)
            Escalation-->>Ops: Escalate khẩn ngay, không chờ dashboard định kỳ
        else Chưa vượt ngưỡng thời gian
            Escalation->>Escalation: Tiếp tục theo dõi
        end
    end
```
