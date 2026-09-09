# Enhance sequence — Circuit breaker & graceful provider switch

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có breaker/provider dự phòng). Đáp ứng yêu cầu thứ 4 của đề bài: khi breaker mở kéo dài, tự động chuyển toàn bộ chuyến đang chạy sang provider dự phòng theo cơ chế graceful, không gián đoạn phiên, kèm thông báo rõ cho tài xế.

```mermaid
sequenceDiagram
    participant Breaker as CIRCUIT_BREAKER_STATE (provider chính)
    participant Primary as Maps Provider chính
    participant Fallback as FALLBACK_PROVIDER_CONFIG store
    participant Assignment as ACTIVE_TRIP_PROVIDER_ASSIGNMENT store
    participant FallbackProvider as Maps Provider dự phòng
    actor Driver

    loop Theo dõi liên tục
        Primary-->>Breaker: Cập nhật tỉ lệ lỗi + soft_timeout theo latency_threshold_ms
    end

    alt Vượt ngưỡng, breaker mở kéo dài
        Breaker->>Breaker: state=open
        Breaker->>Fallback: Lấy fallback_provider có graceful_switch=true
        Breaker->>Assignment: Với toàn bộ TRIP đang active, cập nhật provider_in_use=fallback
        Note over Assignment: Chuyển ngầm trong phiên đang chạy, tài xế không cần khởi động lại app
        Assignment-->>FallbackProvider: Các lệnh tính route tiếp theo route sang provider dự phòng
        FallbackProvider-->>Driver: Route/ETA tiếp tục khả dụng liên tục, không gián đoạn
        Assignment-->>Driver: Hiển thị banner "Đang dùng nguồn bản đồ dự phòng, chất lượng route có thể khác biệt"
    else Breaker vẫn closed
        Breaker->>Breaker: Tiếp tục dùng Primary bình thường
    end
```
