# Sổ cái giao dịch tài chính nội bộ (ledger) yêu cầu linearizability nghiêm ngặt

**Hệ thống:** Hệ thống ledger ghi nhận giao dịch tiền giữa các tài khoản trong một fintech, không cho phép mất hoặc double-apply giao dịch.

**Vai trò của flow:** Consensus đảm bảo mọi giao dịch được ghi theo đúng một thứ tự duy nhất và toàn bộ cluster đồng thuận, kể cả khi node chết hoặc mạng phân vùng.

**Yêu cầu cụ thể:**
- Mỗi giao dịch chỉ được commit khi đã replicate tới majority node, và số dư tài khoản chỉ được cập nhật sau khi log entry commit (state machine apply sau commit, không apply sớm).
- Trong trường hợp network partition, phần thiểu số (minority) phải chuyển sang chế độ read-only/reject-write, không tự ý tạo giao dịch mới để tránh double-ledger khi partition được hàn lại (healed).
- Phải có cơ chế phát hiện và log rõ "split-brain" giả định (hai leader cùng tồn tại do term cũ) và tự động vô hiệu hóa leader cũ ngay khi nó nhận ra term mới hơn.
- Idempotency: mỗi giao dịch có transaction ID duy nhất; nếu client retry do timeout, hệ thống không apply hai lần dù request đã thực sự thành công ở lần gửi trước.
- Đo lường/observability: expose được commit latency p50/p99, số lần leader election trong 24h, và alert khi cluster mất quorum quá X giây.
