# Enhance sequence — Đối soát song song (shadow) giữa bản stable và canary

Đây là **enhance**, flow hoàn toàn mới, chưa tồn tại ở base. Đáp ứng yêu cầu 3 của đề bài: mọi thay đổi liên quan tới tính toán số dư/giao dịch phải chạy shadow — gửi cùng request tới cả bản stable và canary, so sánh kết quả, nhưng chỉ trả về kết quả từ bản đang chính thức — để phát hiện sai lệch trước khi thực sự chuyển traffic sang canary.

```mermaid
sequenceDiagram
    actor Customer
    participant Router as Router
    participant Stable as Deployment stable (chính thức)
    participant Canary as Deployment canary (shadow)
    participant Compare as SHADOW_COMPARISON store
    actor Dev as Kỹ sư triển khai

    Customer->>Router: Gửi request liên quan tới số dư/giao dịch
    Router->>Stable: Xử lý chính thức, đồng bộ
    par Gửi song song, không chờ, không ảnh hưởng response cho khách hàng
        Router->>Canary: Gửi bản sao request tương tự (shadow), bất đồng bộ
    end

    Stable-->>Router: Trả kết quả chính thức (vd số dư = 1.000.000đ)
    Router-->>Customer: Trả kết quả từ Stable, khách hàng không hề biết có shadow chạy song song

    Canary-->>Compare: Trả kết quả tính toán riêng của canary (vd số dư = 999.000đ)
    Compare->>Compare: So sánh stable_result và canary_result cho cùng 1 request

    alt Kết quả khớp nhau
        Compare->>Compare: Ghi SHADOW_COMPARISON(is_mismatch=false)
    else Kết quả lệch nhau
        Compare->>Compare: Ghi SHADOW_COMPARISON(is_mismatch=true, stable_result, canary_result)
        Compare->>Dev: Cảnh báo ngay lập tức, kèm chi tiết request và 2 kết quả lệch nhau
        Note over Dev,Compare: Phát hiện sai lệch qua shadow trước khi canary được cấp thêm traffic thật, tránh đưa số liệu sai tới khách hàng thật
    end
```
