# Enhance sequence — Resharding rollback (không mất phần đã migrate)

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base vì base rehash toàn bộ ngay lập tức, không có khái niệm migration job để rollback. Đáp ứng yêu cầu 4 của đề bài: phát hiện lỗi giữa quá trình migrate và rollback được mà không mất dữ liệu đã migrate một phần.

```mermaid
sequenceDiagram
    participant Job as MIGRATION_JOB
    participant NodeOld as NODE cũ
    participant NodeNew as NODE mới
    participant Ops as Ops Monitor
    participant Metric as RESHARDING_METRIC store

    Job->>NodeNew: Copy nền theo batch, cập nhật checkpoint_key sau mỗi batch thành công
    Job->>Metric: Ghi thời gian bắt đầu, tiến độ theo GB đã migrate

    Ops->>Job: Phát hiện lỗi giữa chừng (ví dụ NodeNew trả lỗi ghi liên tục, hoặc p99 latency cache tăng vọt bất thường)
    Ops->>Job: Yêu cầu rollback MIGRATION_JOB

    Job->>Job: Đọc checkpoint_key gần nhất đã xác nhận migrate thành công
    Job->>NodeOld: Giữ nguyên dữ liệu từ checkpoint trở về trước (chưa từng bị xóa khỏi NodeOld trong lúc dual-write)
    Job->>Job: Đặt rollback_available=false sau khi rollback, hủy phần dữ liệu đã copy dở dang trên NodeNew sau checkpoint
    Job->>Ring: Khôi phục ring về trạng thái trước khi bắt đầu migration cho dải hash này

    Note over NodeOld,NodeNew: Vì NodeOld không bị xóa dữ liệu cho tới khi migration hoàn tất hẳn, rollback chỉ cần hủy phần dở dang trên NodeNew, không mất bất kỳ dữ liệu nào

    Job->>Metric: Ghi nhận thời gian hoàn thành (hoặc rollback) và ảnh hưởng p99 latency trong suốt quá trình, phục vụ đo lường theo yêu cầu 5
    Metric-->>Ops: Báo cáo thời gian resharding cho khối lượng dữ liệu cụ thể và mức tăng p99 latency trong lúc resharding
```
