# Sequence - Enhance - Flow "instance-shutdown"

Đây là **enhance**, cùng flow `instance-shutdown` như ở base nhưng thay đổi hoàn toàn cách xử lý, đáp ứng yêu cầu 1, 2 và 3 trong đề bài: thay vì một grace period cố định ngắn cắt hết mọi kết nối, instance tính grace period theo phân vị thời lượng video (yêu cầu 2), phân loại từng session theo thời lượng còn lại (yêu cầu 1), và khi buộc phải ngắt thì gửi tín hiệu rõ ràng để player biết cần reconnect đúng vị trí (yêu cầu 3), thay vì để kết nối treo hoặc timeout im lặng như ở base.

```mermaid
sequenceDiagram
    actor Client
    participant Instance
    participant Policy as Grace Period Policy
    participant Orchestrator as Autoscaler / Deploy tool
    participant LB as Load Balancer
    participant NewInstance as Instance khác

    Orchestrator->>Instance: SIGTERM (yêu cầu shutdown)
    Instance-->>Instance: ngừng nhận connection mới

    Instance->>Policy: lấy grace period đã tính (ví dụ đủ cho p95 stream tự kết thúc)
    Policy-->>Instance: grace_period_seconds

    Instance-->>Instance: phân loại từng stream session đang mở\ntheo estimated_remaining_seconds

    par Session sắp kết thúc tự nhiên (còn lại dưới grace period)
        Instance-->>Client: tiếp tục gửi chunk bình thường
        Note over Instance,Client: video kết thúc tự nhiên trước khi grace period hết,\ndisconnect_reason = completed_naturally
    and Session còn rất dài (vượt quá grace period)
        Instance-->>Instance: sinh resume_token, lưu position_at_disconnect
        Instance-->>Client: đóng kết nối kèm tín hiệu rõ ràng,\nví dụ header X-Reconnect và resume_token,\ndisconnect_reason = shutdown_forced
        Client->>LB: yêu cầu reconnect kèm resume_token
        LB->>NewInstance: điều hướng sang instance còn sống
        NewInstance-->>Client: tiếp tục phát đúng vị trí đã dừng, không hiển thị lỗi
    end

    Instance-->>Orchestrator: xác nhận đã drain xong, có thể tắt process
```
