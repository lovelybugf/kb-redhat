# Cài đặt và nâng cấp

> Cài Satellite (connected/disconnected), Capsule, load balancer, chứng chỉ, nâng cấp.

## Trang

<!-- AUTO-INDEX START -->
- [Capsule sau load balancer](capsule-load-balancer.md) — Đặt hai hay nhiều Capsule sau một load balancer TCP (không SSL offload) để đăng ký host và cấp nội dung vẫn chạy khi một Capsule chết.
- [Capsule installer lỗi 401 OAuth khi đăng ký với Satellite](capsule-oauth-401.md) — Khi `satellite-installer --scenario capsule` dừng ở bước đăng ký Capsule với Satellite và báo 401, lệnh installer đang mang OAuth consumer key/secret khác cặp Foreman đang dùng, thường sau một lần rotate key.
- [Chứng chỉ SSL của Satellite và Capsule (custom, gia hạn, làm mới CA)](certificates.md) — Satellite mặc định dùng CA tự ký do installer sinh ra; muốn dùng CA doanh nghiệp thì tạo CSR đúng chuẩn và kiểm bằng `katello-certs-check` trước khi triển khai.
- [Cài Capsule Server 6.19](install-capsule.md) — Cài Capsule theo thứ tự: tin CA của Satellite, đăng ký OS bằng Activation Key, cài `satellite-capsule`, sinh tarball bằng `capsule-certs-generate` trên Satellite, chạy lệnh installer mà nó in ra.
- [Cài Satellite Server 6.19 có kết nối Internet (connected)](install-satellite-connected.md) — Cài Satellite connected gồm sáu bước: chuẩn bị RHEL 9 sạch, đăng ký RHSM và bật 4 repo, cài gói `satellite`, chạy `satellite-installer --scenario satellite`, import manifest, rồi kiểm tra bằng `hammer ping`.
- [Cài Satellite Server 6.19 không có Internet (disconnected)](install-satellite-disconnected.md) — Satellite không có Internet được cài từ hai ISO (RHEL 9 Binary DVD và Satellite 6.19 Binary DVD) bằng script `install_packages` thay cho đăng ký RHSM.
- [Nâng cấp Satellite lên 6.19 và cập nhật bản vá 6.19.z](upgrade-satellite.md) — Lên 6.19 chỉ đi từ 6.18: sao lưu, chuẩn bị repo 6.19, chạy `satellite-maintain self-upgrade`, `upgrade check`, `upgrade run`, sync repo 6.19 rồi nâng cấp từng Capsule.
<!-- AUTO-INDEX END -->
