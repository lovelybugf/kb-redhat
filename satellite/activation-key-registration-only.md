---
title: Activation Key chỉ áp dụng lúc đăng ký host
tags: [concept, satellite, activation-key, registration, content-view, capsule]
product: Red Hat Satellite 6.16
source: "Red Hat Satellite 6.16 — Managing content, Chương 9; Managing hosts, mục 2.8, 2.9"
reliability: official
updated: 2026-11-23
owner: lovelybugf
---
**TL;DR:** Activation Key (AK) chỉ được áp dụng một lần, lúc đăng ký host; sửa AK sau đó không làm thay đổi các host đã đăng ký. Muốn đổi lifecycle environment hoặc content view của host đã đăng ký thì sửa trên chính host (Web UI hoặc `hammer host update`); muốn đổi Capsule thì dùng Change content source.

## Bản chất

- AK là bộ giá trị mặc định Satellite gán cho host **tại thời điểm đăng ký** (`subscription-manager register --activationkey ...` hoặc lệnh đăng ký sinh từ Hosts → Register Host): content view environment, trạng thái bật/tắt repository, host collection, release version, system purpose.
- Sau khi đăng ký, host không còn gắn theo AK: thay đổi trên AK chỉ có tác dụng với host đăng ký **sau** thời điểm thay đổi.
- Hệ quả vận hành:
  - Sửa AK = chuẩn bị cho các lần đăng ký mới (host mới hoặc host đăng ký lại).
  - Không dùng việc sửa AK để chuyển host đang chạy sang lifecycle environment/content view khác — host vẫn giữ giá trị cũ.

| Muốn | Làm ở đâu | Host đã đăng ký có đổi? |
|---|---|---|
| Đổi LE/CV mặc định cho host đăng ký sau này | Sửa Activation Key | Không |
| Đổi LE/CV của host đang chạy | Trang host → Edit content view environments, hoặc `hammer host update` | Có |
| Đổi Capsule phục vụ host | Change content source | Có, sau khi áp dụng lệnh trên host |

## Đổi lifecycle environment / content view của host đã đăng ký

### Web UI

1. **Hosts → All Hosts**, bấm tên host.
2. Thẻ **Content view environment** → **Edit content view environments**.
3. Chọn lifecycle environment và content view mới → **Save**.

### Hammer

```bash
hammer host update \
  --name "<HOST_FQDN>" \
  --organization-id <ORG_ID> \
  --content-view-environments "<LIFECYCLE_ENV>/<CONTENT_VIEW>"
```

Gán nhiều content view environment (multi-CV): liệt kê cách nhau bởi dấu phẩy, ví dụ `"Library/<CV_A>,<LIFECYCLE_ENV>/<CV_B>"`.

Kiểm tra trên host sau khi đổi:

```bash
subscription-manager refresh
dnf repolist
```

## Đổi Capsule (content source) của host

1. **Hosts → All Hosts**, bấm tên host → mở menu tuỳ chọn cạnh nút **Edit** → **Change content source**.
2. Chọn Capsule mới, lifecycle environment, content view.
3. Áp dụng trên host: chạy job remote execution mà Satellite đề xuất, hoặc chạy tay trên host các lệnh được sinh tự động trong màn hình Change content source.

Lưu ý: Capsule mới phải được gán lifecycle environment chứa content view đó và đã đồng bộ, nếu không host sẽ không có nội dung.

## Liên quan

- [Download policy của repository](download-policy.md)
- [Tắt Red Hat Lightspeed](disable-lightspeed.md) — global parameter cũng chỉ tác động lúc đăng ký, host cũ phải xử lý riêng.

## Nguồn

- Red Hat Satellite 6.16, Managing content, Chương 9 (activation keys) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.16/html/managing_content/index
- Red Hat Satellite 6.16, Managing hosts, mục 2.8, 2.9 — https://docs.redhat.com/en/documentation/red_hat_satellite/6.16/html/managing_hosts/index
- Cú pháp `hammer host update --content-view-environments` và đường dẫn Change content source đối chiếu thêm với bản 6.19 (Managing content; Managing hosts) ngày 2026-11-23.
