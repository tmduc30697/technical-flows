# Enhance sequence — Network partition handling (migrate hoặc pause, không split-brain)

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base vì base không xử lý network partition. Đáp ứng yêu cầu 4 của đề bài: khi cụm bị chia 2 phía, các trận đang chạy ở phía minority phải được migrate sang majority hoặc tạm dừng, tuyệt đối không để 2 phía cùng xử lý độc lập.

```mermaid
sequenceDiagram
    participant NodeMin as NODE (phía minority)
    participant NodeMaj as NODE (phía majority)
    participant Status as PARTITION_STATUS
    actor Client

    Note over NodeMin,NodeMaj: Network partition xảy ra, cụm chia thành 2 phía không liên lạc được với nhau

    Client->>NodeMin: Gửi INPUT_EVENT trong lúc đang partition
    NodeMin->>NodeMin: Thử replicate CONSENSUS_LOG entry mới
    NodeMin--xNodeMaj: Không liên lạc được với đủ node khác để đạt majority
    Note over NodeMin: NodeMin không thể commit log mới vì không đạt quorum, đây là guarantee cốt lõi ngăn split-brain

    NodeMin->>Status: Ghi PARTITION_STATUS(match_id, partition_side=minority)
    alt Chiến lược migrate (chấp nhận gián đoạn ngắn)
        Status->>NodeMaj: Yêu cầu tiếp quản match dựa trên MATCH_STATE đã commit gần nhất phía majority
        NodeMaj->>NodeMaj: Khởi tạo lại trận từ state đã commit, trở thành leader mới cho match
        NodeMaj-->>Client: Thông báo "Đang kết nối lại" rồi tiếp tục trận trên phía majority
        NodeMin->>NodeMin: Dừng hẳn xử lý match này ở phía minority
    else Chiến lược pause
        Status->>NodeMin: Đánh dấu action=pause cho match
        NodeMin-->>Client: Thông báo "Trận tạm dừng, đang chờ kết nối mạng phục hồi"
        Note over NodeMin: Không xử lý input mới nào cho tới khi partition được hàn lại
    end

    Note over NodeMin,NodeMaj: Dù chọn chiến lược nào, tuyệt đối chỉ một phía được tiếp tục xử lý match tại một thời điểm, tránh 2 kết quả khác nhau cho cùng 1 trận
```
