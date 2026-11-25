# Satellite

> Red Hat Satellite: quản lý nội dung (repository, content view, lifecycle environment), đăng ký host, Capsule, provisioning, Lightspeed.

- **Phiên bản đang dùng:** 6.19 (các dự án hiện tại). Trang trích nguồn phiên bản cũ hơn ghi rõ ở trường `product`.
- **Tài liệu chính thức:** [Red Hat Satellite 6.19 — docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_satellite/6.19)
- Placeholder dùng chung: `<SATELLITE_FQDN>`, `<CAPSULE_FQDN>`, `<HOST_FQDN>`, `<ORG>`, `<ORG_ID>`, `<LIFECYCLE_ENV>`, `<CONTENT_VIEW>`, `<REPO_ID>`.

## Bắt đầu từ đâu

| Câu hỏi | Trang |
|---|---|
| Satellite gồm những gì, mở cổng nào? | [Thành phần và dịch vụ](architecture/components-and-services.md), [Cổng mạng](architecture/network-ports.md) |
| Dựng mới Satellite/Capsule | [Cài Satellite (connected)](installation/install-satellite-connected.md), [Cài Capsule](installation/install-capsule.md) |
| Đưa nội dung tới host | [Repository và đồng bộ](content/repositories-and-sync.md) → [Content View và Lifecycle](content/content-views-and-lifecycle.md) |
| Đăng ký host, vá lỗi | [Đăng ký host](hosts/host-registration.md) → [Vá bằng errata](hosts/patching-with-errata.md) |
| Sao lưu, sự cố | [Backup và restore](operations/backup-and-restore.md), [Health check và xử lý sự cố](operations/health-check-and-troubleshooting.md) |
| Tra lệnh nhanh | [Hammer cheatsheet](ansible/hammer-cheatsheet.md) |

## Trang

<!-- AUTO-INDEX START -->
- [Ansible và Hammer](ansible/_index.md) — Tích hợp Ansible trong Satellite, Satellite Ansible Collection, tra cứu nhanh lệnh hammer.
- [Kiến trúc và triển khai](architecture/_index.md) — Thành phần Satellite/Capsule, cổng mạng, mô hình triển khai, sizing và tuning.
- [Quản lý nội dung](content/_index.md) — Subscription/manifest, repository và đồng bộ, download policy, content view và lifecycle environment, errata, đồng bộ giữa các Satellite, virt-who.
- [Quản lý host](hosts/_index.md) — Đăng ký host, host group và tham số, remote execution, vá errata, gói, nâng cấp RHEL, report template, Lightspeed.
- [Cài đặt và nâng cấp](installation/_index.md) — Cài Satellite (connected/disconnected), Capsule, load balancer, chứng chỉ, nâng cấp.
- [Vận hành](operations/_index.md) — Sao lưu và khôi phục, health check và xử lý sự cố, cứu host khi Satellite hỏng, dung lượng, thông báo và webhook, giám sát.
- [Provisioning](provisioning/_index.md) — Cấp phát máy: template, PXE, boot disk và kickstart không cần DHCP, discovery, compute resource VMware và cloud.
- [Bảo mật](security/_index.md) — OpenSCAP, xác thực LDAP/AD/IdM, người dùng và phân quyền, che bí mật trong tài liệu.
<!-- AUTO-INDEX END -->
