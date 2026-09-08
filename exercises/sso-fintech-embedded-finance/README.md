# Fintech cho phép SSO giữa web, mobile app và đối tác nhúng (embedded finance)

**Hệ thống:** Một nền tảng fintech cung cấp dịch vụ thanh toán, có web app, mobile app riêng, và cho phép đối tác nhúng (embed) trải nghiệm đăng nhập vào app của đối tác (embedded finance).

**Vai trò của flow:** Nền tảng vừa là IdP cho web/mobile app của chính mình, vừa cấp một luồng SSO hạn chế cho đối tác thứ ba nhúng qua iframe/webview mà không lộ thông tin đăng nhập gốc.

**Yêu cầu cụ thể:**
- Session SSO giữa web và mobile app phải đồng bộ trạng thái đăng nhập (đăng xuất trên web phải phản ánh gần như ngay lập tức trên mobile) qua cơ chế push/token revocation, không chỉ chờ token tự hết hạn.
- Luồng nhúng cho đối tác phải giới hạn phạm vi truy cập (không cấp toàn quyền như đăng nhập trực tiếp), và không cho phép đối tác thấy hoặc lưu lại thông tin đăng nhập gốc của người dùng.
- Phải có Single Logout xuyên suốt: người dùng đăng xuất ở một nơi phải kết thúc session ở cả web, mobile và mọi phiên nhúng ở đối tác liên quan.
- Có cơ chế phát hiện và ngăn brute-force/token theft trong quá trình SSO (ví dụ nhiều lần thử với state/nonce sai từ cùng một IP).
- Ghi log audit đầy đủ cho mọi phiên SSO liên quan đến tiền (ai, từ thiết bị nào, qua kênh nào) để đáp ứng yêu cầu compliance tài chính.
