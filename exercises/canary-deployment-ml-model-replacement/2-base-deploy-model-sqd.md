# Base sequence — Deploy model (thay thế trực tiếp, chỉ đo kỹ thuật)

Đây là **base**, flow "Thay model mới" ở trạng thái hiện tại — chuyển thẳng 100% traffic, chỉ xét ổn định kỹ thuật. Flow này liên quan mật thiết tới enhance vì toàn bộ 5 yêu cầu của đề bài đều nhằm cải tổ lại đúng flow này.

```mermaid
sequenceDiagram
    participant MLOps as MLOps Engineer
    participant Old as Model cũ (đang serving)
    participant New as Model mới đã train
    participant Serving as Serving Layer
    participant Dashboard as Technical Dashboard (latency/lỗi)

    MLOps->>Serving: Deploy Model mới, thay thế Model cũ
    Serving->>Serving: 100% traffic chuyển sang Model mới ngay lập tức
    Serving-->>Dashboard: Theo dõi latency, tỷ lệ lỗi kỹ thuật
    Dashboard-->>MLOps: Latency tốt, không có lỗi kỹ thuật
    MLOps->>MLOps: Kết luận "deploy thành công" chỉ dựa trên chỉ số kỹ thuật
    Note over Old: Model cũ bị gỡ hoàn toàn, không còn traffic nào để so sánh song song
    Note over Dashboard,MLOps: Không đo tỷ lệ click/chuyển đổi mua hàng theo từng model, không biết chất lượng gợi ý thực tế có tốt hơn hay tệ hơn
```
