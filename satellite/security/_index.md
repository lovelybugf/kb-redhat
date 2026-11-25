# Bảo mật

> OpenSCAP, xác thực LDAP/AD/IdM, người dùng và phân quyền, che bí mật trong tài liệu.

## Trang

<!-- AUTO-INDEX START -->
- [Bí mật cần che khi đưa ảnh chụp và lệnh Satellite vào tài liệu](documentation-secrets.md) — Ảnh chụp và lệnh lấy từ quá trình triển khai Satellite hay lộ ba loại bí mật: mật khẩu admin, OAuth consumer key/secret, và token trong lệnh global registration.
- [Xác thực ngoài cho Satellite với LDAP, Active Directory, IdM và Keycloak](external-authentication.md) — Satellite nhận năm nguồn xác thực ngoài: LDAP (chỉ mật khẩu), Active Directory nối trực tiếp và IdM (thêm Kerberos SSO), Red Hat build of Keycloak và Red Hat SSO (thêm OTP/TOTP).
- [Quét tuân thủ bảo mật với OpenSCAP trên Satellite](openscap-compliance.md) — OpenSCAP trên Satellite gồm bốn bước: bật plugin OpenSCAP trên Capsule, nạp SCAP content khớp phiên bản RHEL của host, tạo compliance policy, rồi triển khai policy lên host bằng Ansible, Puppet hoặc thủ công.
- [Organization, Location, người dùng và phân quyền RBAC trong Satellite](users-roles-orgs.md) — Organization chia tài nguyên theo đơn vị sở hữu (manifest, content, host), Location nhóm theo vị trí vật lý và có phân cấp; user chỉ thấy và gán được tài nguyên trong Org/Location của mình.
<!-- AUTO-INDEX END -->
