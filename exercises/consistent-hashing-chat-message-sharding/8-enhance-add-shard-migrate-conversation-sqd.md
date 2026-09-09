# Enhance sequence — Add shard, migrate conversation an toàn

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base vì base không có consistent hashing nên không có khái niệm "chỉ di chuyển phần bị dịch chuyển". Đáp ứng yêu cầu 3 (thêm shard chỉ ảnh hưởng conversation thuộc range bị dịch chuyển, không downtime) và yêu cầu 4 (tin nhắn gửi trong lúc migrate không bị mất hoặc ghi nhầm shard cũ) của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant App as Chat App Server
    participant Ring as HASH_RING
    participant ShardOld as SHARD cũ (from_shard)
    participant ShardNew as SHARD mới (to_shard)
    participant Migration as SHARD_MIGRATION

    Note over Ring: Thêm SHARD mới vào cụm, chỉ các virtual node trong 1 dải hash bị dịch chuyển sang shard mới, các conversation khác không đổi
    Ring->>Migration: Xác định conversation C1 rơi vào dải bị dịch chuyển, tạo SHARD_MIGRATION(from=cũ, to=mới, status=in_progress, dual_write_active=true)

    Migration->>ShardOld: Đọc toàn bộ MESSAGE của C1 để copy nền (background copy)
    ShardOld-->>Migration: Trả dữ liệu

    par Trong lúc copy nền vẫn có tin nhắn mới gửi tới
        User->>App: Gửi tin nhắn mới vào C1
        App->>Migration: Kiểm tra C1 đang migrate (dual_write_active=true)
        App->>ShardOld: Ghi tin nhắn vào shard cũ (vẫn là nguồn xác thực trong lúc copy)
        App->>ShardNew: Đồng thời ghi (dual-write) vào shard mới
    end

    Migration->>Migration: Copy nền hoàn tất, xác nhận ShardNew đã có đủ dữ liệu cũ cộng dữ liệu ghi trong lúc migrate
    Migration->>Ring: Cập nhật hash ring, conversation_id=C1 chính thức trỏ tới SHARD mới
    Migration->>Migration: Đặt status=completed, dual_write_active=false
    Migration->>ShardOld: Xóa dữ liệu C1 khỏi shard cũ

    Note over App,ShardNew: Trong suốt quá trình, request đọc/ghi của user vẫn được phục vụ bình thường, không downtime, và không tin nhắn nào bị mất hoặc ghi nhầm shard cũ sau khi cutover
```
