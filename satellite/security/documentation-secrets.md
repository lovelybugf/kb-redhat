---
title: Bí mật cần che khi đưa ảnh chụp và lệnh Satellite vào tài liệu
tags: [reference, satellite, security]
product: Red Hat Satellite 6.19
source: "Phiên Claude Code 2026-11-25 (551a81b5)"
reliability: internal-tested
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Ảnh chụp và lệnh lấy từ quá trình triển khai Satellite hay lộ ba loại bí mật: mật khẩu admin, OAuth consumer key/secret, và token trong lệnh global registration. Che trên bản sao (giữ ảnh gốc ở nơi lưu của dự án), rồi rà lại tài liệu đầu ra để tìm bí mật còn sót.

## Chỗ hay lộ

| Bí mật | Thường nằm trong |
|---|---|
| Mật khẩu admin Web UI | Màn hình kết thúc `satellite-installer`, lệnh `hammer` có tham số mật khẩu, file cấu hình hammer của user |
| OAuth consumer key/secret | Màn hình Administer → Settings (Authentication), output cài đặt |
| Token đăng ký host | Lệnh global registration sinh từ Hosts → Register Host (`curl ... -H 'Authorization: Bearer <TOKEN>'`) |

Ngoài ra luôn thay tên khách hàng, Organization/Location, FQDN, IP, tên tài khoản bằng placeholder theo [chuẩn kb-core](https://github.com/lovelybugf/kb-core/blob/main/standards/page-structure.md).

## Quy trình che

1. Làm trên bản sao; ảnh gốc giữ ở thư mục dự án, không đưa vào kho public.
2. Che bằng khối màu đặc, không làm mờ (ảnh làm mờ có thể khôi phục được).
3. Lệnh trong tài liệu: thay giá trị bằng `<ADMIN_PASSWORD>`, `<OAUTH_SECRET>`, `<REGISTRATION_TOKEN>`.
4. Rà tài liệu cuối: tìm chuỗi `Bearer`, `password`, `oauth`, `secret` trong phần chữ, và xem lại từng ảnh.

## Liên quan

- [Huỷ đăng ký và đăng ký lại host](../hosts/host-reregistration.md)

## Nguồn

- Phiên Claude Code 2026-11-25 (551a81b5): che 7 ảnh chụp triển khai chứa ba loại bí mật trên, rồi rà lại file đầu ra.
