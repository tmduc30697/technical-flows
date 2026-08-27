# Livestream bán hàng với chốt đơn tức thời

**Hệ thống:** Một nền tảng livestream bán hàng, người bán giới thiệu sản phẩm trực tiếp, viewer đặt mua ngay trong lúc xem, giá/khuyến mãi/tồn kho có thể thay đổi liên tục ngay trong buổi live.

**Vai trò của flow:** Nhận ingest từ người bán, transcode/phân phối tới viewer, đồng thời đảm bảo các mốc sự kiện chốt giá/mở bán hiển thị trên luồng được đóng dấu thời gian chính xác để hệ thống đặt hàng đối chiếu đúng, bất kể độ trễ phân phối khác nhau giữa các viewer.

**Yêu cầu cụ thể:**
- Độ trễ ingest-tới-viewer khác nhau giữa các viewer (do CDN edge, chất lượng mạng) có thể khiến viewer xem chậm hơn nhìn thấy giá/tồn kho đã lỗi thời tại thời điểm đặt hàng — cần đóng dấu thời gian sự kiện chốt giá/mở bán ngay tại nguồn phát để hệ thống order xác thực theo trạng thái thực tại thời điểm phát sóng, không theo thời điểm viewer bấm mua trên máy của họ.
- Khi người bán bị rớt kết nối đột ngột giữa lúc nhiều viewer đang chuẩn bị chốt đơn, hệ thống phải phân biệt được "gián đoạn tạm thời" và "kết thúc phiên live" để tránh đóng phiên bán hàng quá sớm khiến các đơn đang xử lý dở bị hủy oan.
- Overlay số lượng tồn kho hiển thị trên luồng video luôn có độ trễ vài giây so với hệ thống tồn kho thực — hệ thống đặt hàng không được tin tưởng tuyệt đối vào con số hiển thị trên overlay mà phải luôn đối chiếu lại tồn kho thực tại thời điểm xử lý đơn, đồng thời không để overlay lệch quá xa gây viewer bức xúc khi đặt hàng thất bại.
- Thời điểm mở bán flash sale trong live có thể khiến lượng viewer tăng đột biến trong vài giây, kéo tải lên cả pipeline ingest/CDN lẫn hệ thống đặt hàng downstream — cần cơ chế cách ly để tải đột biến phía checkout không làm ảnh hưởng ngược lại tới độ ổn định của luồng video đang phát cho những viewer khác.
- Khi buổi live được ghi lại làm VOD phát lại sau, các mốc thời gian giới thiệu từng sản phẩm phải được lưu chính xác đồng bộ với luồng gốc, tránh tình huống viewer xem VOD bấm mua vào sản phẩm/giá đã hết hiệu lực kể từ buổi live gốc.
