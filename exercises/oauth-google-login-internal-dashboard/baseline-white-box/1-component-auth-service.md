# Component — Baseline Auth Service

**Lớp: white-box** — **Nhóm: Hiện trạng**. Diagram này tự thiết kế breakdown bên trong hệ thống cho phần xác thực baseline: tách Auth Service thành 2 trách nhiệm rõ ràng (kiểm tra credential, phát session) để làm nền so sánh khi Google OAuth được thêm vào ở `change-white-box/`. Component không cần đề bài nêu rõ, tự đề xuất thiết kế hợp lý.

<svg viewBox="0 0 720 320" width="100%" xmlns="http://www.w3.org/2000/svg">
<rect x="20" y="120" width="160" height="80" fill="none" stroke="#333" stroke-width="2" rx="4"/>
<text x="100" y="155" font-family="sans-serif" font-size="13" fill="#333" text-anchor="middle">Dashboard</text>
<text x="100" y="172" font-family="sans-serif" font-size="13" fill="#333" text-anchor="middle">Frontend</text>
<rect x="270" y="40" width="220" height="240" fill="none" stroke="#333" stroke-width="2" rx="4"/>
<text x="380" y="60" font-family="sans-serif" font-size="12" fill="#333" text-anchor="middle" font-weight="bold">Auth Service</text>
<rect x="290" y="80" width="180" height="70" fill="none" stroke="#333" stroke-width="1.5" rx="3"/>
<text x="380" y="110" font-family="sans-serif" font-size="12" fill="#333" text-anchor="middle">Credential</text>
<text x="380" y="126" font-family="sans-serif" font-size="12" fill="#333" text-anchor="middle">Validator</text>
<rect x="290" y="190" width="180" height="70" fill="none" stroke="#333" stroke-width="1.5" rx="3"/>
<text x="380" y="220" font-family="sans-serif" font-size="12" fill="#333" text-anchor="middle">Session</text>
<text x="380" y="236" font-family="sans-serif" font-size="12" fill="#333" text-anchor="middle">Issuer (JWT/cookie)</text>
<rect x="580" y="120" width="120" height="80" fill="none" stroke="#333" stroke-width="2" rx="4"/>
<text x="640" y="165" font-family="sans-serif" font-size="13" fill="#333" text-anchor="middle">User DB</text>
<line x1="180" y1="160" x2="268" y2="160" stroke="#333" stroke-width="2" marker-end="url(#arrow1)"/>
<text x="224" y="150" font-family="sans-serif" font-size="11" fill="#333" text-anchor="middle">POST /login</text>
<line x1="380" y1="150" x2="380" y2="188" stroke="#333" stroke-width="1.5" marker-end="url(#arrow1)"/>
<text x="440" y="172" font-family="sans-serif" font-size="10" fill="#333" text-anchor="middle">on success</text>
<line x1="470" y1="115" x2="578" y2="150" stroke="#333" stroke-width="1.5" marker-end="url(#arrow1)"/>
<text x="560" y="115" font-family="sans-serif" font-size="10" fill="#333" text-anchor="middle">query user</text>
<defs>
<marker id="arrow1" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#333"/></marker>
</defs>
</svg>

**Ghi chú thiết kế:** `Credential Validator` chỉ so khớp password hash với `User DB`; `Session Issuer` chịu trách nhiệm phát JWT/cookie riêng — tách 2 trách nhiệm này để phần Thay đổi (thêm Google OAuth) chỉ cần cắm thêm 1 nguồn xác thực mới mà không phải sửa `Session Issuer`.
