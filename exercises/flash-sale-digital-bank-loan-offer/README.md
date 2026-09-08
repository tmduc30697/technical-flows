# Mở bán suất vay ưu đãi lãi suất thấp giới hạn số lượng của ngân hàng số

**Hệ thống:** Một app ngân hàng số mở chương trình vay tiêu dùng lãi suất ưu đãi, giới hạn 1000 suất, khách hàng phải bấm đăng ký ngay khi mở, ai đăng ký trước và đủ điều kiện được duyệt trước.

**Vai trò của flow:** Flow đăng ký phải xử lý đúng thứ tự "đến trước được trước" khi có lượng lớn khách hàng bấm đăng ký cùng lúc, đồng thời phải chạy kiểm tra điều kiện tín dụng (gọi hệ thống bên ngoài, có độ trễ) mà không làm sai thứ tự ưu tiên.

**Yêu cầu cụ thể:**
- Vì kiểm tra điều kiện tín dụng có độ trễ (vài giây, gọi credit bureau), không thể trừ suất ngay khi bấm đăng ký — yêu cầu thiết kế 2 bước: (1) giữ "vị trí hàng đợi" atomic ngay khi bấm đăng ký (theo timestamp/counter tăng dần), (2) xác nhận suất chỉ sau khi qua kiểm tra tín dụng, và nếu không đạt điều kiện phải nhả suất cho người kế tiếp trong hàng đợi.
- Mô tả cụ thể: khách hàng #999 và #1000 giữ được vị trí hợp lệ, nhưng khách #999 bị từ chối tín dụng sau 5 giây — suất của #999 phải được chuyển đúng cho khách #1001 (người xếp hàng kế tiếp), không phải bỏ trống suất đó, và toàn bộ việc "nhả suất - cấp suất kế tiếp" phải atomic để không có 2 khách cùng được cấp 1 suất.
- Đảm bảo bộ đếm vị trí hàng đợi (counter) tăng đúng tuần tự dưới tải cao (dùng cơ chế atomic increment ở DB hoặc cache, không dùng đọc-rồi-tăng ở application vì sẽ mất increment khi nhiều request song song).
- Nếu request kiểm tra tín dụng bị timeout (hệ thống bên ngoài chậm), quy định retry tối đa bao nhiêu lần và trong thời gian bao lâu trước khi coi là thất bại và nhả suất cho người kế tiếp — không để 1 khách timeout chặn vô thời hạn cả hàng đợi phía sau.
- Có dashboard hiển thị real-time số suất còn lại và vị trí hàng đợi hiện tại cho khách đang chờ, đồng bộ đúng với trạng thái thật trong DB (không hiển thị số liệu cache trễ gây hiểu nhầm đã hết suất khi thực ra còn).
