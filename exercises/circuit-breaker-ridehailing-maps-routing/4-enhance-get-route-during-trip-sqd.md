# Enhance sequence — Get route during trip (fallback cache tức thời, retry siêu chặt)

Đây là **enhance**, cùng flow "Get route during trip" đã có ở base nhưng nay thay đổi hoàn toàn theo yêu cầu 1, 2 và 3 của đề bài: coi chậm (soft timeout) là lỗi, fallback ngay về cached route trong lúc chờ, retry tối đa 1-2 lần với backoff cực ngắn.

```mermaid
sequenceDiagram
    actor Driver
    participant App as Driver App
    participant Maps as Maps API (bên thứ ba)
    participant Cache as CACHED_ROUTE store
    participant Breaker as CIRCUIT_BREAKER_STATE

    Note over Driver: Xe đang di chuyển trong chuyến đi đang chạy
    App->>Maps: Gọi tính route/ETA
    App->>Cache: Đồng thời lấy ngay cached_route gần nhất
    Cache-->>App: Trả route đã cache
    App-->>Driver: Hiển thị ngay route cache trong lúc chờ (không bao giờ treo màn hình)

    Maps-->>App: Phản hồi sau X ms
    alt Vượt latency_threshold_ms (soft timeout, coi như lỗi)
        App->>Breaker: Ghi nhận lỗi (status=soft_timeout) để tính vào ngưỡng mở breaker
        App->>Maps: Retry (tối đa 1-2 lần, backoff cực ngắn — vài trăm ms)
        Maps-->>App: Kết quả retry
        alt Retry thành công và còn nhanh
            App->>Cache: Cập nhật cached_route mới
            App-->>Driver: Cập nhật route chính xác hơn
        else Vẫn chậm/lỗi sau retry
            App-->>Driver: Tiếp tục dùng cached_route, chuyển sang flow "Circuit breaker & graceful switch" nếu lặp lại nhiều
        end
    else Phản hồi nhanh, trong ngưỡng
        App->>Cache: Cập nhật cached_route mới
        App-->>Driver: Hiển thị route mới nhất
    end
```
