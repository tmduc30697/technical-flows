# Enhance sequence — Xử lý gián đoạn ingest với cửa sổ khôi phục

Đây là **enhance** của flow `handle-disconnect` đã có ở base. So với base (kết thúc session ngay khi mất kết nối, tạo session mới khi reconnect), flow này thay đổi ở chỗ: ingest phát hiện mất kết nối trong vài giây, chuyển session sang trạng thái `interrupted` (hiển thị "gián đoạn" cho viewer thay vì kết thúc hẳn) và giữ session mở trong 1 khoảng `reconnect_deadline`, streamer reconnect trong khoảng đó sẽ tiếp tục đúng session cũ — đáp ứng yêu cầu 3.

```mermaid
sequenceDiagram
    actor Streamer
    participant Ingest as Ingest Server
    participant Session as Stream Session Store
    actor Viewer

    Streamer--x Ingest: Mat ket noi dot ngot (rot mang)
    Ingest->>Ingest: Phat hien mat heartbeat trong vong vai giay
    Ingest->>Session: UPDATE status = interrupted, interrupted_at = now,<br/>reconnect_deadline = now + N giay
    Ingest-->>Viewer: Hien thi trang thai "gian doan", khong ngat hoan toan playback

    alt Streamer ket noi lai truoc reconnect_deadline
        Streamer->>Ingest: Ket noi lai voi cung stream key
        Ingest->>Session: Doi chieu STREAM_SESSION dang interrupted cua streamer nay
        Ingest->>Session: UPDATE status = live, xoa interrupted_at/reconnect_deadline
        Ingest-->>Viewer: Tu dong khoi phuc playback tren cung session, khong reload
    else Qua reconnect_deadline ma khong ket noi lai
        Ingest->>Session: UPDATE status = ended, ended_at = now
        Ingest-->>Viewer: Bao stream da ket thuc that su
    end
```
