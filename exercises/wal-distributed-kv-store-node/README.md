# Node lưu trữ trong distributed KV store dùng WAL cho từng node cục bộ

**Hệ thống:** Một node trong cụm KV store phân tán (đã có replication ở tầng cluster) vẫn cần WAL cục bộ để đảm bảo node đó tự phục hồi đúng sau crash, không chỉ dựa vào replica khác.

**Vai trò của flow:** WAL cục bộ đảm bảo dữ liệu ghi vào node không bị mất ngay cả khi chỉ một node duy nhất đó crash, giảm tải phải luôn rebuild từ replica (vốn tốn network/thời gian hơn).

**Yêu cầu cụ thể:**
- Node phải tự phục hồi từ WAL cục bộ trước khi thông báo với cluster rằng nó "sẵn sàng trở lại" (rejoin), tránh trường hợp trả dữ liệu cũ/thiếu trong lúc đang catch-up từ replica.
- Nếu WAL cục bộ bị corrupt hoàn toàn (đĩa hỏng), node phải tự phát hiện và chuyển sang chế độ rebuild-from-replica thay vì cố gắng replay dữ liệu hỏng, tránh đưa dữ liệu sai vào cluster.
- Có sự phối hợp rõ ràng giữa recovery cục bộ (WAL) và recovery ở tầng cluster (Raft log) — không được để 2 cơ chế mâu thuẫn nhau về thứ tự áp dụng thay đổi.
- Đo lường: so sánh thời gian một node phục hồi bằng WAL cục bộ (nhanh) versus phải rebuild toàn bộ từ replica qua network (chậm hơn nhiều), để quyết định ngưỡng khi nào nên dùng cách nào.
- Test kịch bản mất điện toàn bộ datacenter khiến nhiều node cùng crash đồng thời — đảm bảo mỗi node phục hồi độc lập đúng trạng thái của nó trước khi cluster đồng thuận lại state chung.
