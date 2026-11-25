---
title: "RHEL 10: dnf báo GPG check FAILED vì thiếu release key 4 (khoá hậu lượng tử)"
tags: [troubleshooting, rhel, security, satellite]
product: RHEL 10.0/10.1, kể cả host do Satellite quản lý
source: "Red Hat KB 3449341; Product Signing Keys (access.redhat.com/security/team/key); Phiên Claude Code 2026-11-25 (dc97f4ce)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Trên RHEL 10, gói được ký bằng release key 4 (khoá hậu lượng tử ML-DSA-87 + Ed448, ID `05707a62`) bị dnf từ chối nếu gói `redhat-release` còn cũ, chưa chứa khoá này. Cách sửa chính thức: cập nhật `redhat-release` trước, kiểm tra khoá đã có, rồi cài lại gói bị chặn; với Satellite, Content View của host phải chứa bản `redhat-release` mới.

## Triệu chứng

- `dnf install`/`dnf update` lỗi "GPG check FAILED", báo các khoá "are already installed but they are not correct for this package".

## Chẩn đoán

```bash
rpm -q redhat-release
rpm -q gpg-pubkey | grep -i 05707a62      # không có kết quả = chưa có release key 4
# Xem tên gói bị chặn (file chỉ có trong cache sau một lần dnf install lỗi):
rpm -qp --qf "%{NAME}\n" /var/cache/dnf/rhel-10-*/packages/<PACKAGE>-*.rpm
```

## Nguyên nhân

Gói `redhat-release` đến bản 10.1-16 chưa chứa release key 4, trong khi các gói phát hành từ khoảng tháng 10/2025 được ký bằng khoá này (theo KB 3449341).

## Cách xử lý

Chạy bằng root:

```bash
dnf update -y redhat-release             # lên 10.1-17 trở lên
rpm -q gpg-pubkey | grep -i 05707a62     # phải có kết quả
dnf install <PACKAGE>                    # cài lại gói bị chặn
```

- Host đăng ký Satellite: sync repo, publish và promote Content View để Lifecycle Environment của host có bản `redhat-release` mới; nếu không, host không lấy được bản chứa khoá.
- Không nhập khoá này bằng `gpg --import` (báo "unknown version 6", KB 7133967); nhập tay thì dùng `rpm --import <FILE>`.
- Chỉ khi chính lệnh cập nhật `redhat-release` cũng bị chặn: phương án tự dựng, **Red Hat chưa xác nhận** — `dnf download redhat-release`, trích file khoá bằng `rpm2cpio`, rồi `rpm --import` file khoá đó.

## Phòng ngừa

- Provisioning template (post-script): chạy `dnf update -y redhat-release` trước `dnf update`.
- Trước đợt vá, đưa `redhat-release` mới lên mọi Lifecycle Environment.

## Liên quan

- [Satellite — mục lục](../satellite/_index.md)

## Nguồn

- Red Hat KB 3449341 (Solution Verified, RHEL 10/10.1) — https://access.redhat.com/solutions/3449341
- Red Hat Product Signing Keys, release key 4 (fingerprint bắt đầu `FCD355B305707A62`) — https://access.redhat.com/security/team/key
- Red Hat KB 7133967 (lỗi gpg "unknown version 6", Solution In Progress) — https://access.redhat.com/solutions/7133967
- Đối chiếu trong phiên Claude Code 2026-11-25 (dc97f4ce); chưa chạy thử trên host RHEL 10 (lab là RHEL 9).
