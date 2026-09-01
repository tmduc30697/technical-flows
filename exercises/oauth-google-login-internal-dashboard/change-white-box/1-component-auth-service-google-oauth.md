# Component — Auth Service with Google OAuth

**Lớp: white-box** — **Nhóm: Thay đổi**. Diagram này mở rộng breakdown của `baseline-white-box/1-component-auth-service.md`: thêm 2 thành phần mới — `Google OAuth Client` (giao tiếp với Google) và `Identity Linker` (account linking/tạo user) — trong khi `Session Issuer` giữ nguyên, không đổi, đúng như thiết kế baseline đã tách sẵn để việc thêm nguồn xác thực mới không phải sửa phần phát session.

<svg viewBox="0 0 780 400" width="100%" xmlns="http://www.w3.org/2000/svg">
<rect x="20" y="165" width="150" height="70" fill="none" stroke="#333" stroke-width="2" rx="4"/>
<text x="95" y="195" font-family="sans-serif" font-size="12" fill="#333" text-anchor="middle">Dashboard</text>
<text x="95" y="211" font-family="sans-serif" font-size="12" fill="#333" text-anchor="middle">Frontend</text>
<rect x="220" y="40" width="280" height="330" fill="none" stroke="#333" stroke-width="2" rx="4"/>
<text x="360" y="60" font-family="sans-serif" font-size="12" fill="#333" text-anchor="middle" font-weight="bold">Auth Service</text>
<rect x="240" y="75" width="240" height="55" fill="none" stroke="#333" stroke-width="1.5" rx="3"/>
<text x="360" y="107" font-family="sans-serif" font-size="12" fill="#333" text-anchor="middle">Credential Validator (baseline)</text>
<rect x="240" y="140" width="240" height="55" fill="none" stroke="#0a5" stroke-width="1.5" stroke-dasharray="4,3" rx="3"/>
<text x="360" y="172" font-family="sans-serif" font-size="12" fill="#0a5" text-anchor="middle">Google OAuth Client (new)</text>
<rect x="240" y="205" width="240" height="55" fill="none" stroke="#0a5" stroke-width="1.5" stroke-dasharray="4,3" rx="3"/>
<text x="360" y="230" font-family="sans-serif" font-size="12" fill="#0a5" text-anchor="middle">Identity Linker (new)</text>
<text x="360" y="246" font-family="sans-serif" font-size="10" fill="#0a5" text-anchor="middle">domain check + link/create user</text>
<rect x="240" y="270" width="240" height="55" fill="none" stroke="#333" stroke-width="1.5" rx="3"/>
<text x="360" y="302" font-family="sans-serif" font-size="12" fill="#333" text-anchor="middle">Session Issuer (baseline, không đổi)</text>
<rect x="580" y="70" width="170" height="65" fill="none" stroke="#333" stroke-width="2" stroke-dasharray="6,4" rx="4"/>
<text x="665" y="98" font-family="sans-serif" font-size="12" fill="#333" text-anchor="middle">Google</text>
<text x="665" y="114" font-family="sans-serif" font-size="11" fill="#333" text-anchor="middle">(external system)</text>
<rect x="580" y="270" width="170" height="65" fill="none" stroke="#333" stroke-width="2" rx="4"/>
<text x="665" y="298" font-family="sans-serif" font-size="12" fill="#333" text-anchor="middle">User DB</text>
<text x="665" y="314" font-family="sans-serif" font-size="10" fill="#333" text-anchor="middle">users + oauth_identities</text>
<line x1="170" y1="200" x2="218" y2="167" stroke="#333" stroke-width="1.5" marker-end="url(#arrow3)"/>
<line x1="170" y1="200" x2="218" y2="167" stroke="#333" stroke-width="1.5"/>
<line x1="170" y1="200" x2="238" y2="167" stroke="#0a5" stroke-width="1.5" marker-end="url(#arrow3g)"/>
<text x="205" y="150" font-family="sans-serif" font-size="10" fill="#0a5" text-anchor="middle">/auth/google/start</text>
<line x1="480" y1="167" x2="578" y2="105" stroke="#0a5" stroke-width="1.5" marker-end="url(#arrow3g)"/>
<text x="560" y="128" font-family="sans-serif" font-size="10" fill="#0a5" text-anchor="middle">authorize/token/userinfo</text>
<line x1="480" y1="230" x2="578" y2="300" stroke="#0a5" stroke-width="1.5" marker-end="url(#arrow3g)"/>
<text x="560" y="270" font-family="sans-serif" font-size="10" fill="#0a5" text-anchor="middle">read/write user, oauth_identity</text>
<line x1="360" y1="195" x2="360" y2="203" stroke="#0a5" stroke-width="1.5" marker-end="url(#arrow3g)"/>
<line x1="360" y1="260" x2="360" y2="268" stroke="#0a5" stroke-width="1.5" marker-end="url(#arrow3g)"/>
<defs>
<marker id="arrow3" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#333"/></marker>
<marker id="arrow3g" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6 Z" fill="#0a5"/></marker>
</defs>
</svg>

**Ghi chú thiết kế:** `Google OAuth Client` chỉ gọi ra ngoài (authorize/token/userinfo) và không tự quyết định tạo user; `Identity Linker` mới là nơi áp domain whitelist + account-linking logic, tách riêng để dễ test và để `Credential Validator` (baseline) không bị đụng chạm.
