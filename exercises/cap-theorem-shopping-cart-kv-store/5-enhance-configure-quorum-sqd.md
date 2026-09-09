# Enhance sequence — Configure quorum (định nghĩa N, W, R có lý giải)

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base có cấu hình quorum nhưng không lý giải). Đáp ứng yêu cầu thứ 1 của đề bài: định nghĩa cụ thể N, W, R cho giỏ hàng và giải thích rõ có chọn W+R>N hay không.

```mermaid
sequenceDiagram
    actor Ops as Platform Engineer
    participant Config as QUORUM_CONFIG store

    Ops->>Config: Đề xuất N=3, W=1, R=1 cho giỏ hàng
    Config->>Config: Tính W+R so với N → 1+1=2 < N=3
    Config->>Config: satisfies_wr_gt_n = false
    Ops->>Config: Ghi rationale — "Giỏ hàng ưu tiên availability và tốc độ hơn strong consistency tức thời, chấp nhận đọc có thể miss ghi gần nhất, xử lý bằng merge khi phát hiện conflict thay vì chặn đọc/ghi"
    Config-->>Ops: Cấu hình được lưu kèm lý giải rõ ràng, không còn là quorum "ngầm hiểu"
    Note over Config: Nếu sau này 1 team khác cần đổi N/W/R, phải cập nhật lại rationale tương ứng — tránh quorum trôi dạt không ai còn nhớ vì sao chọn vậy
```
