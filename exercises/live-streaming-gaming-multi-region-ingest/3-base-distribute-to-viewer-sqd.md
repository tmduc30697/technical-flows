# Base sequence — Phân phối nội dung trực tiếp từ điểm ingest gốc

Đây là **base**, mô tả flow phân phối nội dung ở trạng thái hiện tại: mọi viewer, dù ở khu vực nào, đều kéo luồng trực tiếp từ điểm ingest trung tâm gốc, không có bước nhân bản nội dung ra khu vực gần viewer. Đây là tiền đề cho enhance yêu cầu 3 — nhân bản/phân phối nội dung tới các khu vực khác để viewer ở xa xem với độ trễ hợp lý.

```mermaid
sequenceDiagram
    actor ViewerNear as Viewer gan Central-PoP
    actor ViewerFar as Viewer o khu vuc xa
    participant Central as Central-PoP

    ViewerNear->>Central: Yeu cau xem stream
    Central-->>ViewerNear: Tra ve luong truc tiep, do tre thap

    ViewerFar->>Central: Yeu cau xem stream (keo xuyen luc dia)
    Central-->>ViewerFar: Tra ve luong truc tiep, do tre cao do khoang cach vat ly

    Note over Central,ViewerFar: Khong co ban sao noi dung o khu vuc gan ViewerFar,<br/>moi viewer xa deu phai chiu do tre tuong duong khoang cach toi Central-PoP
```
