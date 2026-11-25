# kb-redhat

> Kho tri thức sản phẩm Red Hat: Satellite, OpenShift, Identity Management, RHEL, Ansible Automation Platform và sản phẩm khác.

Mỗi `_index.md` của sản phẩm ghi phiên bản đang dùng và link tài liệu chính thức trên docs.redhat.com. Kiến thức nền trung lập (Linux, Kubernetes, GitOps) nằm ở kb-base-devops. Chuẩn viết: kb-core.

## Quản lý hạ tầng

- [Satellite](satellite/_index.md) — 6.19, 8 nhóm: kiến trúc, cài đặt/nâng cấp/Capsule, nội dung (CV/LE, errata, sync), host (đăng ký, REX, vá), provisioning, bảo mật (OpenSCAP, LDAP/IdM, RBAC), vận hành (backup, DR, health check), Ansible/hammer.
- [RHEL](rhel/_index.md) — phần riêng sản phẩm RHEL: subscription, Leapp, vòng đời phiên bản.
- [Identity Management (IdM)](idm/_index.md) — người dùng, Kerberos, DNS, HBAC, trust với AD.

## Nền tảng ứng dụng

- [OpenShift Container Platform](ocp/_index.md) — cài đặt, vận hành cụm, Operator, OpenShift GitOps.

## Tự động hoá

- [Ansible và Ansible Automation Platform](ansible-aap/_index.md) — playbook, collection, controller, execution environment.

## Optional

- [Sản phẩm Red Hat khác](other/_index.md) — Quay, ACM, ACS, ... khi chưa đủ trang để tách thư mục.
- [README.md](README.md) — hướng dẫn nhanh cho người.
- [log.md](log.md) — nhật ký thay đổi (chỉ ghi thêm).
- [Chuẩn chung kb-core](https://github.com/lovelybugf/kb-core/blob/main/index.md) — khuôn trang, tag, quy trình.
