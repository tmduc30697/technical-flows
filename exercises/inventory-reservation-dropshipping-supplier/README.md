# Đặt trước hàng dropshipping khi tồn kho thực tế nằm ở nhà cung cấp bên thứ ba

**Hệ thống:** Một sàn dropshipping hiển thị tồn kho lấy từ API của nhiều nhà cung cấp khác nhau, tồn kho thực tế không do sàn kiểm soát trực tiếp mà chỉ đồng bộ định kỳ.

**Vai trò của flow:** Flow reservation phải giữ "chỗ" tạm thời ở tồn kho cache nội bộ khi khách đặt mua, đồng thời xác nhận thật với nhà cung cấp trước khi chốt đơn, xử lý đúng khi tồn kho cache bị lệch so với thực tế.

**Yêu cầu cụ thể:**
- Vì tồn kho từ nhà cung cấp chỉ đồng bộ mỗi vài phút (không real-time), yêu cầu khi khách đặt mua phải: (1) trừ tạm tồn kho cache nội bộ ngay (atomic, tránh 2 khách cùng đặt đơn vị cuối theo cache), (2) gọi API xác nhận đặt hàng thật với nhà cung cấp, (3) nếu nhà cung cấp báo hết hàng thật thì rollback lại cache và báo khách hủy đơn kèm hoàn tiền.
- Mô tả cụ thể race: 2 khách đặt mua sản phẩm mà cache nội bộ báo còn 1, cả 2 đều trừ cache thành công do cache đã lệch thực tế (nhà cung cấp chỉ còn 1 thật) — quy định thứ tự xử lý: request xác nhận với nhà cung cấp nào đến trước được ưu tiên giữ, request sau khi biết nhà cung cấp báo hết phải tự động hủy và hoàn tiền, có thông báo rõ cho khách thứ 2 nêu nguyên nhân (lệch dữ liệu tồn kho từ nhà cung cấp).
- Đồng bộ tồn kho từ nhà cung cấp về cache nội bộ phải là update atomic không được ghi đè mất các thay đổi tạm thời (reservation đang giữ) tạo ra bởi các đơn đang xử lý song song trong lúc đồng bộ diễn ra.
- Khi API nhà cung cấp timeout ở bước xác nhận đặt hàng thật, quy định retry tối đa bao nhiêu lần trong bao lâu trước khi coi là thất bại và hoàn tiền tự động cho khách — không để đơn hàng ở trạng thái "đang xử lý" quá lâu mà không có hành động rõ ràng.
- Có cơ chế cảnh báo khi tỷ lệ hủy đơn do lệch tồn kho với 1 nhà cung cấp cụ thể vượt ngưỡng (dấu hiệu đồng bộ dữ liệu với nhà cung cấp đó có vấn đề), để team vận hành có thể tạm ẩn sản phẩm của nhà cung cấp đó khỏi sàn.
