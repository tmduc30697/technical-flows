# Nền tảng streaming giải trí kiểu Twitch

**Hệ thống:** Một nền tảng cho phép streamer phát trực tiếp gameplay/nội dung giải trí tới đông đảo người xem.

**Vai trò của flow:** Nhận luồng RTMP/SRT từ phần mềm phát của streamer, transcode real-time ra nhiều bitrate và phân phối qua CDN tới người xem.

**Yêu cầu cụ thể:**
- Ingest server phải xác thực stream key trước khi nhận luồng, từ chối ngay các kết nối dùng key sai/đã bị revoke, không cho tạo luồng "ma".
- Transcode real-time ra tối thiểu 3 mức bitrate (nguồn, trung, thấp) để hỗ trợ adaptive bitrate streaming, độ trễ thêm vào không vượt quá vài giây so với luồng gốc.
- Nếu kết nối ingest từ streamer bị rớt giữa buổi (mất mạng), hệ thống phải phát hiện trong vòng vài giây và tự động chuyển stream sang trạng thái "gián đoạn" hiển thị cho viewer, tự khôi phục khi streamer kết nối lại mà không tạo stream session mới.
- Toàn bộ segment video trong lúc live phải được lưu tạm để phục vụ ghép VOD sau khi kết thúc, không được để mất segment khi có sự cố worker transcode.
- Kiến trúc phân phối phải chịu được lượng viewer đồng thời lớn tăng đột biến (raid, streamer nổi tiếng lên live) mà không làm tăng độ trễ cho toàn bộ viewer khác.
