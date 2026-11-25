# Kiến trúc và triển khai

> Thành phần Satellite/Capsule, cổng mạng, mô hình triển khai, sizing và tuning.

## Trang

<!-- AUTO-INDEX START -->
- [Thành phần, dịch vụ và mô hình kết nối của Satellite](components-and-services.md) — Satellite Server là Foreman cộng plugin Katello (Candlepin lo subscription, Pulp lo content) trên nền PostgreSQL, Redis, httpd; Capsule là Smart Proxy (`foreman-proxy`) cộng Pulp, đặt gần host.
- [Lập kế hoạch triển khai Satellite](deployment-planning.md) — Trước khi cài, chốt mô hình content, số Capsule theo vị trí và vùng mạng, cấu trúc Organization/Location, đường Lifecycle Environment và các quyết định không đổi được sau cài như FIPS.
- [Cổng mạng và chiều kết nối của Satellite, Capsule và host](network-ports.md) — Host chỉ cần tới Capsule của mình qua 443, 80 và 9090 (thêm 8000, 1883, 8140, 67–69 khi dùng tính năng tương ứng); chỉ Satellite đi ra Red Hat CDN qua 443.
- [Sizing phần cứng, dung lượng và tuning hiệu năng Satellite](sizing-and-tuning.md) — Tối thiểu Satellite cần 4 core, 20 GB RAM, 4 GB swap và khoảng 300 GB cho `/var/lib/pulp`; Capsule cần 4 core, 12 GB RAM.
<!-- AUTO-INDEX END -->
