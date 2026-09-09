# Enhance sequence — Hot key handling (chia nhỏ đếm cục bộ, tổng hợp định kỳ)

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base vì base route toàn bộ traffic của 1 key vào đúng 1 node theo consistent hashing thuần, không xử lý hot key. Đáp ứng yêu cầu 4 của đề bài: phát hiện hot key và chia nhỏ việc đếm ra nhiều node để tránh nghẽn cổ chai, đánh đổi độ chính xác tức thời lấy khả năng chịu tải.

```mermaid
sequenceDiagram
    participant Detector as Hot Key Detector
    participant NodeA as NODE A (phụ trách chính theo ring)
    participant NodeB as NODE B (đếm cục bộ hộ)
    participant NodeC as NODE C (đếm cục bộ hộ)
    participant HotKeyStore as HOT_KEY_PARTIAL_COUNTER

    loop Định kỳ theo dõi QPS theo từng key
        Detector->>NodeA: Lấy QPS hiện tại của từng API key qua NodeA
        NodeA-->>Detector: API key K có QPS cực lớn, vượt ngưỡng hot key
    end

    Detector->>Detector: Đánh dấu HOT_KEY(K), bật chế độ chia nhỏ đếm
    Note over NodeA,NodeC: Traffic của K nay được router chia đều theo round-robin cho NodeA, NodeB, NodeC thay vì dồn hết vào 1 node

    par Mỗi node đếm cục bộ phần traffic của mình
        NodeA->>HotKeyStore: Ghi HOT_KEY_PARTIAL_COUNTER(K, node=A, partial_count)
        NodeB->>HotKeyStore: Ghi HOT_KEY_PARTIAL_COUNTER(K, node=B, partial_count)
        NodeC->>HotKeyStore: Ghi HOT_KEY_PARTIAL_COUNTER(K, node=C, partial_count)
    end

    loop Tổng hợp định kỳ (ví dụ mỗi vài trăm ms)
        Detector->>HotKeyStore: Cộng dồn partial_count của cả 3 node cho K
        HotKeyStore-->>Detector: Tổng gần đúng hiện tại của K trong window
        alt Tổng gần đúng vượt limit_per_window
            Detector->>NodeA: Phát tín hiệu chặn tạm thời traffic mới của K trên cả 3 node
        else Vẫn trong ngưỡng
            Detector->>Detector: Tiếp tục cho phép, chấp nhận sai số nhỏ giữa các lần tổng hợp
        end
    end
```
