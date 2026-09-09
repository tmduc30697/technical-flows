# Enhance sequence — Ingest ổn định dài giờ qua luân phiên worker

Đây là **enhance**, flow hoàn toàn mới so với base (base không có cơ chế luân phiên worker, ngầm định 1 worker chạy suốt buổi). Đáp ứng yêu cầu 1 trong đề bài: buổi học kéo dài nhiều giờ liên tục phải vận hành ổn định, tránh rò rỉ tài nguyên/tích luỹ lỗi nhỏ theo thời gian, mà không cần restart giữa chừng làm gián đoạn giờ học.

```mermaid
sequenceDiagram
    participant Supervisor as Ingest Supervisor
    participant WorkerA as INGEST_WORKER_INSTANCE A (dang chay)
    participant WorkerB as INGEST_WORKER_INSTANCE B (worker moi)
    participant Session as Class Session Store

    Note over WorkerA: Da chay lien tuc nhieu gio, bat dau tich luy memory/handle nho le

    loop Dinh ky kiem tra suc khoe (vd moi 30 phut)
        Supervisor->>WorkerA: Kiem tra chi so tai nguyen (memory, file handle, latency)
    end

    alt Chi so vuot nguong an toan hoac den chu ky luan phien dinh ky
        Supervisor->>WorkerB: Khoi tao worker moi, ket noi song song vao cung luong dang ingest
        WorkerB->>Session: Dong bo lecture_position_ms va trang thai hien tai tu WorkerA
        Supervisor->>WorkerA: Chuyen giao toan bo ket noi dang xu ly sang WorkerB (handoff khong ngat luong)
        Supervisor->>WorkerA: Retire WorkerA (retired_at = now, restart_reason = rolling-rotation)
        Note over WorkerB,Session: Qua trinh chuyen giao khong lam gian doan luong bai giang dang phat cho hoc sinh
    else Chi so binh thuong
        Supervisor->>WorkerA: Tiep tuc giu nguyen, khong can thay the
    end
```
