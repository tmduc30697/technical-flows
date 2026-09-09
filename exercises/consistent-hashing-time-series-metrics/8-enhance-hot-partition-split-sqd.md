# Enhance sequence — Phát hiện hot partition và tách partition riêng

Đây là **enhance**, flow hoàn toàn mới, chưa tồn tại ở base. Đáp ứng yêu cầu 5 của đề bài: phát hiện 1 service có tải lớn bất thường so với các service khác trong cùng `hash_range`, rồi tách riêng 1 partition dành cho service đó (partition splitting theo nhu cầu).

```mermaid
sequenceDiagram
    participant Monitor as Load Monitor
    participant Stats as SERVICE_LOAD_STATS
    participant Part as PARTITION (hash_range=A, chứa nhiều service)
    participant Splitter as Partition Splitter
    participant NewPart as PARTITION mới (is_dedicated=true, riêng cho Service X)
    participant Event as PARTITION_SPLIT_EVENT

    loop Định kỳ mỗi vài phút
        Monitor->>Stats: Ghi nhận points_per_sec theo từng service trong window gần nhất
    end
    Monitor->>Stats: So sánh points_per_sec của Service X với trung bình các service khác
    alt Service X vượt ngưỡng lệch tải nghiêm trọng
        Stats->>Stats: Đánh dấu is_hot=true cho Service X
        Monitor->>Splitter: Yêu cầu tách partition riêng cho Service X
        Splitter->>Part: Xác định dữ liệu thuộc Service X trong partition hiện tại
        Splitter->>NewPart: Tạo partition mới, is_dedicated=true, gán node riêng
        Splitter->>Part: Di chuyển dữ liệu Service X sang NewPart, còn lại các service khác vẫn ở Part
        Splitter->>Event: Ghi PARTITION_SPLIT_EVENT (source_partition_id, hot_service_id, new_partition_id)
        Note over Part,NewPart: Từ nay ghi/đọc của Service X đi thẳng vào NewPart, không còn tranh chấp tài nguyên với các service khác trong Part
    else Tải vẫn trong ngưỡng bình thường
        Stats-->>Monitor: Không cần tách partition
    end
```
