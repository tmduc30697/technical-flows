# Enhance sequence — Cách ly kênh tương tác khỏi luồng video khi lớp đông

Đây là **enhance**, flow hoàn toàn mới so với base (base xử lý tương tác chung tài nguyên với hệ thống, không có cơ chế cách ly). Đáp ứng yêu cầu 5 trong đề bài: khi lớp học lớn hàng trăm/nghìn học sinh, lượng sự kiện tương tác gửi lên gần như đồng thời không được cạnh tranh tài nguyên với luồng video chính.

```mermaid
sequenceDiagram
    actor Students as Hang tram hoc sinh (lop dong, dang soi noi)
    participant Gateway as Interaction Ingest Gateway (rieng)
    participant Queue as Interaction Event Queue (buffer rieng)
    participant InteractionSvc as Interaction Service (auto-scale rieng)
    participant VideoPipeline as Video Ingest/Transcode/CDN Pipeline

    par Lan song su kien tuong tac gan nhu dong thoi
        Students->>Gateway: Gui hang loat raise_hand/question cung luc
        Gateway->>Queue: Day vao hang doi rieng, tra ve ACK ngay lap tuc cho client
    and Luong video van tiep tuc song song, tren ha tang tach biet
        VideoPipeline->>VideoPipeline: Tiep tuc ingest/transcode/phan phoi binh thuong
    end

    loop InteractionSvc xu ly dan theo kha nang, khong bi don ep
        Queue->>InteractionSvc: Lay batch su kien tu hang doi
        InteractionSvc->>InteractionSvc: Gan lecture_timestamp_ms, luu INTERACTION_EVENT
    end

    Note over Gateway,VideoPipeline: Gateway/Queue/InteractionSvc dung chung nhom ha tang rieng voi VideoPipeline,<br/>ap luc ghi nhan tang dot bien o day khong lam giam chat luong hay tang do tre phat song
```
