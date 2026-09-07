# Canary deployment cho việc thay thế mô hình machine learning đang serving

**Hệ thống:** Một hệ thống gợi ý sản phẩm (recommendation) dùng mô hình machine learning để serving kết quả real-time cho hàng triệu request mỗi ngày, cần thay mô hình mới định kỳ.

**Vai trò của flow:** Khác với deploy code thông thường, ở đây "phiên bản mới" là một mô hình đã train, cần đánh giá dựa trên chất lượng kết quả gợi ý (business metric) không chỉ đúng/sai kỹ thuật.

**Yêu cầu cụ thể:**
- Mô hình mới phải được canary trên một phần nhỏ traffic thật, và kết quả gợi ý phải được đo bằng các metric nghiệp vụ (tỷ lệ click, tỷ lệ chuyển đổi mua hàng) so sánh với mô hình cũ đang chạy song song trên phần traffic còn lại, không chỉ đo latency/lỗi kỹ thuật.
- Xử lý trường hợp mô hình mới hoạt động kỹ thuật hoàn toàn ổn định (không lỗi, latency tốt) nhưng chất lượng gợi ý kém hơn mô hình cũ về mặt nghiệp vụ — vẫn phải coi đây là tiêu chí rollback, không chỉ dựa vào lỗi hệ thống.
- Đảm bảo việc phân chia traffic vào canary của mô hình mới không làm sai lệch dữ liệu dùng để train các mô hình tương lai (tránh feedback loop — mô hình mới ảnh hưởng hành vi user rồi dữ liệu đó lại dùng để train tiếp gây thiên lệch không kiểm soát).
- Thiết kế thời gian canary đủ dài để loại trừ yếu tố thời điểm (ví dụ hành vi mua hàng khác nhau giữa ngày thường và cuối tuần) trước khi kết luận mô hình mới tốt hơn hay kém hơn, tránh quyết định dựa trên mẫu dữ liệu quá ngắn/thiên lệch.
- Có khả năng rollback tức thời về mô hình cũ nếu phát hiện mô hình mới serving ra kết quả bất thường/không phù hợp (ví dụ gợi ý sản phẩm không liên quan hoặc vi phạm chính sách nội dung) mà không cần chờ phân tích đầy đủ metric dài hạn.
