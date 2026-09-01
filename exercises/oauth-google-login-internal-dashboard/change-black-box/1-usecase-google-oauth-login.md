# Use Case — Add Google OAuth Login

**Lớp: black-box** — **Nhóm: Thay đổi**. Diagram này thể hiện năng lực mới được thêm vào: nhân viên đăng nhập bằng Google, và trường hợp nhân viên từ chối cấp quyền ở màn hình consent của Google — đúng 2 yêu cầu đề bài nêu rõ ("thêm nút Sign in with Google" và "xử lý được case user bấm Deny"). "Deny Google consent" được mô hình là `<<extend>>` của "Login with Google" vì nó là 1 nhánh rẽ do chính actor chọn tại màn hình của Google, có luồng xử lý callback khác hẳn (không có `code`, có `error=access_denied`) nên xứng đáng có sequence riêng.

<svg viewBox="0 0 780 330" width="100%" xmlns="http://www.w3.org/2000/svg">
<circle cx="70" cy="110" r="16" fill="none" stroke="#333" stroke-width="2"/>
<line x1="70" y1="126" x2="70" y2="185" stroke="#333" stroke-width="2"/>
<line x1="35" y1="145" x2="105" y2="145" stroke="#333" stroke-width="2"/>
<line x1="70" y1="185" x2="40" y2="230" stroke="#333" stroke-width="2"/>
<line x1="70" y1="185" x2="100" y2="230" stroke="#333" stroke-width="2"/>
<text x="70" y="252" font-family="sans-serif" font-size="13" fill="#333" text-anchor="middle">Employee</text>
<circle cx="710" cy="110" r="16" fill="none" stroke="#333" stroke-width="2"/>
<line x1="710" y1="126" x2="710" y2="185" stroke="#333" stroke-width="2"/>
<line x1="675" y1="145" x2="745" y2="145" stroke="#333" stroke-width="2"/>
<line x1="710" y1="185" x2="680" y2="230" stroke="#333" stroke-width="2"/>
<line x1="710" y1="185" x2="740" y2="230" stroke="#333" stroke-width="2"/>
<text x="710" y="252" font-family="sans-serif" font-size="13" fill="#333" text-anchor="middle">Google</text>
<text x="710" y="268" font-family="sans-serif" font-size="13" fill="#333" text-anchor="middle">(OAuth Provider)</text>
<rect x="180" y="30" width="420" height="260" fill="none" stroke="#333" stroke-width="2" rx="6"/>
<text x="390" y="16" font-family="sans-serif" font-size="14" fill="#333" text-anchor="middle" font-weight="bold">Internal Task Dashboard — Google OAuth (Change)</text>
<ellipse cx="390" cy="105" rx="165" ry="42" fill="none" stroke="#333" stroke-width="2"/>
<text x="390" y="100" font-family="sans-serif" font-size="14" fill="#333" text-anchor="middle">Login with Google</text>
<text x="390" y="118" font-family="sans-serif" font-size="11" fill="#333" text-anchor="middle">(authorization code flow)</text>
<ellipse cx="390" cy="225" rx="165" ry="42" fill="none" stroke="#333" stroke-width="2"/>
<text x="390" y="220" font-family="sans-serif" font-size="14" fill="#333" text-anchor="middle">Deny Google consent</text>
<text x="390" y="238" font-family="sans-serif" font-size="11" fill="#333" text-anchor="middle">(user từ chối cấp quyền)</text>
<line x1="390" y1="183" x2="390" y2="147" stroke="#333" stroke-width="1.5" stroke-dasharray="5,4" marker-end="url(#arrow2)"/>
<text x="440" y="168" font-family="sans-serif" font-size="11" fill="#333" text-anchor="middle">&lt;&lt;extend&gt;&gt;</text>
<line x1="105" y1="145" x2="225" y2="110" stroke="#333" stroke-width="2"/>
<line x1="105" y1="150" x2="225" y2="220" stroke="#333" stroke-width="2"/>
<line x1="675" y1="145" x2="555" y2="110" stroke="#333" stroke-width="2"/>
<defs>
<marker id="arrow2" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#333"/></marker>
</defs>
</svg>

**Actors:**
- **Employee** — nhân viên nội bộ, người trực tiếp bấm "Sign in with Google" và chọn Allow/Deny tại màn hình consent.
- **Google (OAuth Provider)** — hệ thống ngoài, đóng vai trò Authorization Server, tham gia trực tiếp vào luồng "Login with Google" (redirect, cấp `code`, trả `access_token`, trả profile).

**Actions:**
1. **Login with Google** — happy path: authorize → consent Allow → callback → domain check → account link/create → phát session riêng.
2. **Deny Google consent** (`<<extend>>` của #1) — nhân viên bấm Deny tại màn hình Google, hệ thống phải xử lý callback có `error=access_denied` mà không tạo session.
