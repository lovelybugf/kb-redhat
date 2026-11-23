# Satellite

> Red Hat Satellite: quản lý nội dung (repository, content view, lifecycle environment), đăng ký host, Capsule, provisioning, Lightspeed.

- **Phiên bản đang dùng:** 6.19 (các dự án hiện tại). Trang trích nguồn phiên bản cũ hơn ghi rõ ở trường `product`.
- **Tài liệu chính thức:** [Red Hat Satellite 6.19 — docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_satellite/6.19)
- Placeholder dùng chung: `<SATELLITE_FQDN>`, `<CAPSULE_FQDN>`, `<HOST_FQDN>`, `<ORG>`, `<ORG_ID>`, `<LIFECYCLE_ENV>`, `<CONTENT_VIEW>`, `<REPO_ID>`.

## Trang

<!-- AUTO-INDEX START -->
- [Activation Key chỉ áp dụng lúc đăng ký host](activation-key-registration-only.md) — Activation Key (AK) chỉ được áp dụng một lần, lúc đăng ký host; sửa AK sau đó không làm thay đổi các host đã đăng ký.
- [Tắt Red Hat Lightspeed (Insights) trên Satellite và host](disable-lightspeed.md) — Đặt global parameter boolean `host_registration_insights` và `host_registration_insights_inventory` bằng `false` để host đăng ký mới không cài và không gửi dữ liệu Lightspeed.
- [Download policy của repository (on_demand và immediate)](download-policy.md) — Mặc định repo Red Hat dùng `on_demand` (chỉ tải metadata, gói được tải khi host/Capsule yêu cầu), repo custom dùng `immediate`.
<!-- AUTO-INDEX END -->
