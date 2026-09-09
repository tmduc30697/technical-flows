# Enhance sequence — Lưu segment bền vững phục vụ ghép VOD

Đây là **enhance**, flow hoàn toàn mới so với base (base không lưu trữ segment lâu dài, chỉ đẩy thẳng qua CDN cho live). Đáp ứng yêu cầu 4 trong đề bài: toàn bộ segment trong lúc live phải được lưu tạm để ghép VOD sau, và không được mất segment khi 1 worker transcode gặp sự cố giữa chừng.

```mermaid
sequenceDiagram
    participant Transcode as Transcode Worker
    participant Storage as Durable Segment Storage
    participant SegmentDB as Video Segment Store
    participant VOD as VOD Assembler

    loop Moi vai giay, sinh 1 segment moi cho tung rendition
        Transcode->>SegmentDB: Tao VIDEO_SEGMENT status=pending, sequence_no=n
        Transcode->>Storage: Ghi segment len storage ben vung (khong phai disk local cua worker)
        Storage-->>Transcode: Ghi thanh cong
        Transcode->>SegmentDB: UPDATE status = persisted, persisted_at = now
    end

    Note over Transcode: Worker gap su co giua chung (crash, OOM)
    Transcode--xSegmentDB: Mat ket noi dot ngot

    Note over SegmentDB: Cac segment da persisted truoc do van con nguyen trong Storage,<br/>chi segment dang xu ly do (chua kip ghi) bi danh dau lost

    par Worker moi duoc thay the tiep tuc
        Transcode->>SegmentDB: Worker moi lay sequence_no lon nhat da persisted
        Transcode->>Storage: Tiep tuc ghi segment tiep theo tu diem do
    end

    Note over VOD: Sau khi live ket thuc
    VOD->>SegmentDB: Doc toan bo VIDEO_SEGMENT status=persisted theo thu tu sequence_no
    VOD->>Storage: Tai va ghep thanh file VOD hoan chinh
    Note over VOD,Storage: Cac segment bi danh dau lost duoc bao cao rieng,<br/>khong lam gian doan viec ghep phan con lai cua VOD
```
