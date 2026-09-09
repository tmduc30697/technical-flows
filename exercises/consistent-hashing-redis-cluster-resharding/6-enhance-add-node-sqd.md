# Enhance sequence — Add node (chỉ ~1/(N+1) key di chuyển, virtual node tránh hot node)

Đây là **enhance**, cùng flow "add-node" đã có ở base nhưng nay thay đổi hoàn toàn: nhờ consistent hashing với virtual node, thêm 1 node vào cụm N node chỉ khiến khoảng 1/(N+1) tổng số key phải di chuyển, và tải được rải đều nhờ nhiều virtual node cho mỗi node vật lý thay vì dồn vào 1 điểm duy nhất. Đáp ứng yêu cầu 1 và 2 của đề bài.

```mermaid
sequenceDiagram
    participant Ring as HASH_RING
    participant NodeNew as NODE mới thêm vào
    participant Job as MIGRATION_JOB
    participant Test as Test đo tỉ lệ key di chuyển

    Ring->>Ring: Sinh V virtual node cho NODE mới, rải đều nhiều điểm trên ring thay vì chỉ 1 điểm
    Ring->>Ring: Chỉ các virtual node nằm ngay trước mỗi điểm mới bị ảnh hưởng, các dải hash còn lại giữ nguyên chủ sở hữu
    Ring->>Job: Tạo MIGRATION_JOB cho từng dải hash bị dịch chuyển, dual_write_active=true

    loop Với từng dải bị ảnh hưởng
        Job->>Job: Copy nền dữ liệu từ node cũ sang NODE mới
        Job->>Job: Đánh checkpoint_key sau mỗi batch đã copy xong (phục vụ rollback nếu cần)
    end

    Job->>Ring: Toàn bộ dải bị ảnh hưởng copy xong, cập nhật ring, tắt dual_write_active
    Test->>Ring: Đếm số key thực sự đổi node so với tổng số key trước khi thêm node
    Ring-->>Test: Tỉ lệ đo được xấp xỉ 1/(N+1), khớp với lý thuyết consistent hashing
    Note over Ring,NodeNew: Nhờ virtual node rải đều, NODE mới không bị dồn nhận trọn 1 dải hash lớn duy nhất mà nhận nhiều dải nhỏ từ nhiều node khác nhau
```
