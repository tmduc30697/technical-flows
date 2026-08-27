# SSO giữa hệ thống bệnh viện trung tâm và mạng lưới phòng khám liên kết

**Hệ thống:** Một hệ thống bệnh viện trung tâm cung cấp hồ sơ bệnh án điện tử (EHR) dùng chung cho một mạng lưới các phòng khám liên kết độc lập về vận hành nhưng cần truy cập chung một nguồn dữ liệu bệnh nhân.

**Vai trò của flow:** Bệnh viện trung tâm đóng vai trò Identity Provider, cho phép nhân viên y tế ở các phòng khám liên kết SSO vào hệ thống hồ sơ bệnh án trung tâm, đồng thời phải đáp ứng yêu cầu compliance nghiêm ngặt về audit truy cập dữ liệu bệnh nhân.

**Yêu cầu cụ thể:**
- Mỗi lần truy cập hồ sơ bệnh nhân qua phiên SSO phải được ghi log gắn liền với danh tính nhân viên y tế cụ thể, không chỉ gắn với phòng khám, cùng với hồ sơ bệnh nhân nào được xem, để đáp ứng yêu cầu truy vết "ai xem hồ sơ của ai, khi nào, vì lý do gì" khi có thanh tra hoặc khiếu nại.
- Quyền truy cập hồ sơ bệnh nhân qua SSO phải giới hạn theo mối quan hệ điều trị thực tế, tức là nhân viên phòng khám chỉ xem được hồ sơ bệnh nhân đang hoặc đã điều trị tại phòng khám đó, không phải cứ SSO thành công là xem được toàn bộ hồ sơ trong hệ thống trung tâm.
- Khi một phòng khám liên kết chấm dứt hợp đồng với bệnh viện trung tâm, toàn bộ quyền SSO của nhân viên phòng khám đó phải bị thu hồi ngay lập tức và đồng loạt, kể cả với những nhân viên đang có session hoạt động, để tránh truy cập trái phép hồ sơ bệnh nhân sau khi quan hệ đã chấm dứt.
- Phải xử lý được tình huống khẩn cấp lâm sàng, ví dụ bệnh nhân chuyển viện gấp từ phòng khám lên bệnh viện trung tâm cần chia sẻ hồ sơ ngay, bằng một luồng truy cập mở rộng có kiểm soát và giới hạn thời gian, tách biệt khỏi quyền truy cập SSO thông thường và luôn kèm log để review sau.
- Vì thông tin sức khỏe là dữ liệu đặc biệt nhạy cảm, phiên SSO cho nhân viên y tế cần thời gian sống ngắn hơn và bắt buộc xác thực lại thường xuyên hơn so với hệ thống nội bộ thông thường, đặc biệt trên các máy trạm dùng chung tại phòng khám qua nhiều ca kíp nhân viên khác nhau, để tránh một nhân viên vô tình thao tác dưới session còn sót lại của người trước.
