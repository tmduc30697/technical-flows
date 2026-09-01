# Use Case — Baseline Authentication (Email/Password)

**Lớp: black-box** — **Nhóm: Hiện trạng**. Diagram này thể hiện đầy đủ vòng đời xác thực mà hệ thống hiện có trước khi thêm Google OAuth: đăng ký, đăng nhập, đăng xuất bằng email/password. Đề bài chỉ nêu rõ "hiện chỉ có đăng nhập bằng email/password", nhưng 1 hệ thống login-bằng-password thực tế luôn đi kèm Register (để có account) và Logout (để kết thúc session) — thiếu 2 action này thì baseline không đủ để làm nền so sánh thật với nhóm Thay đổi.

<svg viewBox="0 0 720 430" width="100%" xmlns="http://www.w3.org/2000/svg">
<circle cx="70" cy="170" r="16" fill="none" stroke="#333" stroke-width="2"/>
<line x1="70" y1="186" x2="70" y2="245" stroke="#333" stroke-width="2"/>
<line x1="35" y1="205" x2="105" y2="205" stroke="#333" stroke-width="2"/>
<line x1="70" y1="245" x2="40" y2="290" stroke="#333" stroke-width="2"/>
<line x1="70" y1="245" x2="100" y2="290" stroke="#333" stroke-width="2"/>
<text x="70" y="312" font-family="sans-serif" font-size="13" fill="#333" text-anchor="middle">Employee</text>
<text x="70" y="328" font-family="sans-serif" font-size="13" fill="#333" text-anchor="middle">(Internal Staff)</text>
<rect x="230" y="30" width="460" height="370" fill="none" stroke="#333" stroke-width="2" rx="6"/>
<text x="460" y="15" font-family="sans-serif" font-size="14" fill="#333" text-anchor="middle" font-weight="bold">Internal Task Dashboard — Authentication (Baseline)</text>
<ellipse cx="470" cy="110" rx="190" ry="42" fill="none" stroke="#333" stroke-width="2"/>
<text x="470" y="105" font-family="sans-serif" font-size="14" fill="#333" text-anchor="middle">Register</text>
<text x="470" y="123" font-family="sans-serif" font-size="14" fill="#333" text-anchor="middle">(email / password)</text>
<ellipse cx="470" cy="210" rx="190" ry="42" fill="none" stroke="#333" stroke-width="2"/>
<text x="470" y="205" font-family="sans-serif" font-size="14" fill="#333" text-anchor="middle">Login</text>
<text x="470" y="223" font-family="sans-serif" font-size="14" fill="#333" text-anchor="middle">(email / password)</text>
<ellipse cx="470" cy="310" rx="190" ry="42" fill="none" stroke="#333" stroke-width="2"/>
<text x="470" y="316" font-family="sans-serif" font-size="14" fill="#333" text-anchor="middle">Logout</text>
<line x1="105" y1="205" x2="280" y2="110" stroke="#333" stroke-width="2"/>
<line x1="105" y1="205" x2="280" y2="210" stroke="#333" stroke-width="2"/>
<line x1="105" y1="205" x2="280" y2="310" stroke="#333" stroke-width="2"/>
</svg>

**Actor:** Employee (nhân viên nội bộ, thao tác trực tiếp trên các action account/session).

**Actions (3):**
1. **Register (email/password)** — tự tạo account mới, tiền đề để có gì đó mà Login xác thực vào.
2. **Login (email/password)** — action đề bài nêu rõ là hiện trạng.
3. **Logout** — kết thúc session, đối xứng bắt buộc với Login (mọi hệ thống có login-session đều cần logout để thu hồi quyền truy cập).
