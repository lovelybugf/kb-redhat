# Ansible và Hammer

> Tích hợp Ansible trong Satellite, Satellite Ansible Collection, tra cứu nhanh lệnh hammer.

## Trang

<!-- AUTO-INDEX START -->
- [Quản lý cấu hình host bằng Ansible role trong Satellite](ansible-integration.md) — Satellite dùng Ansible để giữ cấu hình host: import role từ Capsule, gán role (và ghi đè biến) cho host hoặc host group, rồi chạy Run Ansible roles.
- [Tra cứu nhanh lệnh hammer theo nhóm tác vụ](hammer-cheatsheet.md) — Bảng tra lệnh `hammer` hay dùng trên Satellite 6.19, nhóm theo tác vụ từ org/location, content, host, errata, job tới RBAC và OpenSCAP, cú pháp lấy từ tài liệu hãng kèm guide và trang PDF nguồn.
- [Tự động hoá cấu hình Satellite bằng collection redhat.satellite](satellite-ansible-collection.md) — Collection `redhat.satellite` (gói `ansible-collection-redhat-satellite`, có sẵn trên Satellite) là bộ module gọi REST API để khai báo Organization, repository, Content View, Activation Key, host bằng playbook.
<!-- AUTO-INDEX END -->
