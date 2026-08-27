# Bộ đếm follower và quan hệ follow/unfollow cho mạng xã hội

**Hệ thống:** Mạng xã hội có tính năng follow/unfollow, và hiển thị số lượng follower trên profile cho hàng triệu user cùng lúc.

**Vai trò của flow:** Tunable consistency cho phép số đếm follower hiển thị nhanh (đọc gần đúng, eventual consistency chấp nhận sai lệch nhỏ tạm thời) trong khi thao tác follow/unfollow thực tế (quan hệ ai đang follow ai) phải chính xác để không gây lỗi logic nghiêm trọng hơn (như hiển thị nhầm quan hệ hoặc để user bấm follow lặp lại).

**Yêu cầu cụ thể:**
- Số đếm follower hiển thị trên profile được phép đọc từ counter tổng hợp bất đồng bộ (cập nhật dần theo batch hoặc CRDT counter), chấp nhận sai lệch tạm thời vài giây tới vài phút so với số thực tế, vì đây chỉ là con số hiển thị không ảnh hưởng logic nghiệp vụ khác.
- Ngược lại, hành động follow/unfollow (bản ghi quan hệ user A follow user B) phải ghi với write quorum đủ cao để đảm bảo ngay sau khi API follow trả thành công, mọi truy vấn tiếp theo (ví dụ kiểm tra "đã follow chưa" để hiển thị nút follow/following) đọc đúng trạng thái mới, tránh hiển thị sai nút bấm khiến user bấm follow nhiều lần.
- Race condition cụ thể: user bấm follow rồi unfollow rất nhanh liên tiếp (double-tap do UI lag) — 2 request này có thể tới 2 node khác nhau gần như đồng thời, cần cơ chế xác định thứ tự đúng (timestamp/version theo từng cặp quan hệ follow) để trạng thái cuối cùng phản ánh đúng ý định cuối của user, tránh "trạng thái không xác định" (đang follow nhưng số đếm coi như đã unfollow).
- Trong lúc network partition giữa các node lưu quan hệ follow, phải quyết định rõ: ưu tiên availability cho hành động follow (chấp nhận ghi ở phía minority, dung hòa/merge khi partition hàn lại theo kiểu "follow thắng unfollow nếu trùng thời điểm không xác định được") hay từ chối ghi nếu không đạt quorum — nêu rõ lựa chọn và hệ quả trải nghiệm user tương ứng.
- Đo lường: độ lệch giữa số đếm follower hiển thị (approximate counter) và số thực tế đếm trực tiếp từ bảng quan hệ (ground truth), chạy đối soát định kỳ và tự động điều chỉnh lại counter nếu độ lệch vượt ngưỡng cho phép.
