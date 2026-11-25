---
title: Repository Red Hat, custom repository và đồng bộ nội dung
tags: [howto, satellite, pulp, hammer]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Managing content, chương 3, 4, 10, phụ lục A (PDF tr. 20–41, 101–106, 172–173); RH403 chương 2, 5; kinh nghiệm triển khai thực tế"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Bật repository Red Hat theo phụ lục A (BaseOS + AppStream đúng bản RHEL, thêm Kickstart nếu provisioning), tạo product riêng cho nội dung ngoài Red Hat kèm GPG key, rồi gắn sync plan chạy hằng ngày. Repository hỏng thì Complete Sync (`--skip-metadata-check true`) hoặc Verify Content Checksum, trên Capsule dùng `hammer capsule content verify-checksum`.

## Product và repository

- **Product** gom các repository liên quan; product Red Hat được tạo tự động khi bật repository đầu tiên, product custom do admin tạo (ví dụ EPEL, ứng dụng nội bộ).
- **Repository** chứa nội dung một loại (yum, file, docker, ansible collection...) theo OS/kiến trúc/release; sync về **Library** của Organization.
- Thực hành tốt: mỗi product và mỗi content view chỉ một loại nội dung; không upload gói vào repository đã có Upstream URL (sync sau có thể ghi đè tùy mirroring policy) — tạo repository riêng không có Upstream URL để upload; tự động hóa tạo product/repository bằng Hammer hoặc Ansible.
- Tắt (disable) repository Red Hat không xóa dữ liệu khỏi đĩa vì Pulp dùng chung nội dung giữa các Organization.

## Repository bắt buộc (phụ lục A)

| Host | Để đăng ký và quản lý | Thêm khi provisioning bằng Kickstart |
|---|---|---|
| RHEL 10 | Red Hat Enterprise Linux 10 for x86_64 - BaseOS (RPMs), - AppStream (RPMs) | ... - BaseOS (Kickstart), - AppStream (Kickstart) |
| RHEL 9 | Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs), - AppStream (RPMs) | ... - BaseOS (Kickstart), - AppStream (Kickstart) |
| RHEL 8 | Red Hat Enterprise Linux 8 for x86_64 - BaseOS (RPMs), - AppStream (RPMs) | ... - BaseOS (Kickstart), - AppStream (Kickstart) |
| RHEL 7 | Red Hat Enterprise Linux 7 Server (RPMs) | Red Hat Enterprise Linux 7 Server (Kickstart) |

Kiến trúc khác thì thay `x86_64` trong tên. Thực tế triển khai còn bật thêm **Red Hat Satellite Client 6** cho từng bản RHEL (công cụ client như katello-host-tools), và repository Capsule/Maintenance đúng phiên bản Satellite nếu muốn Capsule lấy gói từ chính Satellite. Kickstart repository chỉ dùng cho provisioning, không đưa vào content view.

Chọn release khi bật: release major (`9`) là repository "rolling", luôn có gói mới nhất của RHEL 9; release minor (`9.6`) khóa ở bản minor (EUS cần subscription tương ứng). Nếu content view chứa repository minor thì host phải khóa release cùng giá trị (xem phần lỗi hay gặp).

## Bật repository Red Hat

Web UI: **Content → Red Hat Repositories** → chọn loại (RPM, Kickstart...) → tìm tên hoặc bật **Recommended Repositories** → mở repository set → bấm Enable ở dòng kiến trúc/release cần.

```bash
hammer product list --organization "<ORG>"
hammer repository-set list \
  --product "Red Hat Enterprise Linux for x86_64" \
  --organization "<ORG>"
hammer repository-set enable \
  --name "Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)" \
  --releasever "9" --basearch "x86_64" \
  --product "Red Hat Enterprise Linux for x86_64" \
  --organization "<ORG>"
```

Điều kiện: manifest đã import và có subscription ([subscription manifest](subscriptions-manifest.md)); nếu ra Internet qua proxy thì đã khai báo default HTTP proxy.

## Content credentials: SSL và GPG key

- Upstream dùng HTTPS với CA riêng hoặc cần chứng chỉ client: tạo content credential loại SSL Certificate.
- Repository custom có gói ký: import GPG key của nhà cung cấp; có thể gắn ở product (mọi repository dùng chung) hoặc ở repository (ghi đè). Repository Red Hat đã có sẵn khóa, không quản lý GPG key cho chúng.
- Repository ký bằng nhiều khóa: dán tất cả khóa vào cùng một content credential, mỗi khóa cách nhau một dòng.

```bash
# Trích khóa từ gói release của nhà cung cấp, không cần cài gói
rpm2cpio example-release.noarch.rpm | cpio -idmv
ls etc/pki/rpm-gpg/

hammer content-credentials create \
  --content-type gpg_key \
  --name "<GPG_KEY_NAME>" \
  --organization "<ORG>" \
  --path ./etc/pki/rpm-gpg/<KEY_FILE>

hammer content-credentials create \
  --content-type cert \
  --name "<SSL_CERT_NAME>" \
  --organization "<ORG>" \
  --path ~/<CA_CERT_FILE>
```

## Product và repository custom

```bash
hammer product create --name "<PRODUCT>" --organization "<ORG>"
hammer repository create \
  --product "<PRODUCT>" --organization "<ORG>" \
  --name "<REPO_NAME>" --content-type yum \
  --url "<UPSTREAM_URL>" \
  --gpg-key-id <GPG_KEY_ID> \
  --download-policy immediate \
  --publish-via-http true
```

Tùy chọn đáng chú ý khi tạo repository yum (Web UI: **Content → Products → product → New Repository**):

| Tùy chọn | Ghi chú |
|---|---|
| Restrict to Architecture / OS Version (`--arch`, `--os-version`) | Chỉ host khớp mới thấy repository; ưu tiên cao nhất, không override được |
| Upstream URL | Hỗ trợ `http://`, `https://`, `file://` (thư mục phải nằm dưới `/var/lib/pulp/sync_imports/`); bỏ trống để upload thủ công |
| Ignore SRPMs / Ignore treeinfo | Bỏ gói nguồn; bỏ treeinfo khi gặp lỗi "Treeinfo file should have INI format" (mất file Kickstart) |
| Download Policy, Mirroring Policy, Retain package versions | Xem [download policy](download-policy.md) và [mirroring policy](alternate-content-sources.md) |
| Unprotected | Mặc định publish qua HTTP; bỏ tick để bắt buộc chứng chỉ entitlement |
| HTTP Proxy Policy, SSL CA/Client cert/key, Upstream username/password/token | Cho upstream cần proxy hoặc xác thực |

**Repo Discovery** (Content → Products → Repo Discovery) quét một URL và tạo hàng loạt repository trong một product. Upload gói khi không sync được:

```bash
hammer repository upload-content --id <REPO_ID> --path /path/to/package.rpm
hammer repository upload-content --id <REPO_ID> --content-type srpm --path /path/to/package.src.rpm
# File lớn trên mạng chậm bị timeout: tăng Sync task timeout
hammer settings set --name foreman_tasks_sync_task_timeout --value <SECONDS>
```

Repository custom unprotected trong Default Organization View được publish tại `https://<CAPSULE_FQDN>/pulp/content/<ORG_LABEL>/Library/custom/<PRODUCT_LABEL>/<REPO_LABEL>/`.

## Mạng cho đồng bộ

- HTTP proxy: **Infrastructure → HTTP Proxies → New HTTP Proxy**, có thể đặt làm default cho content. Proxy phải cho phép `subscription.rhsm.redhat.com:443`, `cdn.redhat.com:443` (thêm các tên miền console/cert-api nếu dùng Lightspeed) và **không** SSL inspection với hai tên miền đầu.

```bash
hammer http-proxy create --name "<PROXY_NAME>" --url http://<PROXY_FQDN>:<PORT> --content-default-http-proxy true
# Chính sách proxy theo repository: none | global_default_http_proxy | use_selected_http_proxy
hammer repository update --id <REPO_ID> --http-proxy-policy global_default_http_proxy
```

- Upstream dùng cổng lạ: thêm cổng vào SELinux `http_port_t` (mặc định cho phép 80, 81, 443, 488, 8008, 8009, 8443, 9000). Chạy bằng root:

```bash
semanage port -l | grep ^http_port_t
semanage port -a -t http_port_t -p tcp <PORT>
```

## Đồng bộ ad hoc

Web UI: **Content → Products** → chọn repository → **Sync Now**; theo dõi ở **Content → Sync Status** và **Monitor → Satellite Tasks**. Lần sync đầu một bản RHEL lớn có thể mất nhiều giờ nếu băng thông thấp (bảng ước lượng ở mục 4.2: 6 GB mất khoảng 1,5 giờ ở 10 Mbps).

```bash
hammer product synchronize --name "<PRODUCT>" --organization "<ORG>"
hammer repository synchronize --name "<REPO_NAME>" --product "<PRODUCT>" --organization "<ORG>"
# Sync mọi repository của Organization, chạy nền
for i in $(hammer --no-headers --csv repository list --organization "<ORG>" --fields Id); do
  hammer repository synchronize --id "$i" --organization "<ORG>" --async
done
```

## Sync plan

- Sync thường xuyên (hằng ngày) tốt hơn sync dồn hằng tháng: mỗi lần tải ít, tải lên Satellite thấp.
- Nhiều product lớn: tạo nhiều sync plan với custom cron để rải tác vụ qua nhiều giờ; tránh trùng khung giờ backup.

```bash
hammer sync-plan create --name "<SYNC_PLAN>" --organization "<ORG>" \
  --interval daily --sync-date "2026-01-01 01:00:00" --enabled true
# Lịch tùy biến: --interval "custom cron" --cron-expression "30 2 * * *"
hammer product set-sync-plan --name "<PRODUCT>" --sync-plan "<SYNC_PLAN>" --organization "<ORG>"
hammer sync-plan list --organization "<ORG>"
hammer product list --organization "<ORG>" --sync-plan "<SYNC_PLAN>"
```

Nội dung mới chỉ vào Library; host và Capsule chỉ nhận sau khi content view được publish/promote.

## Đồng bộ nâng cao và khôi phục nội dung hỏng

| Tình huống | Cách xử lý | Hammer |
|---|---|---|
| Sync thường, bỏ qua gói không đổi | Optimized Sync | `hammer repository synchronize --id <REPO_ID>` |
| Thiếu gói dù upstream có; sync lỗi giữa chừng | Complete Sync (bỏ kiểm tra metadata) | `... --id <REPO_ID> --skip-metadata-check true` |
| Lỗi 404 với vài gói, "Package does not match intended download" (chỉ yum) | Verify Content Checksum: sync rồi kiểm checksum, tải lại gói sai | `... --id <REPO_ID> --validate-contents true` |
| Host không dùng được gói từ Capsule | Verify checksum trên Capsule theo CV, LE, repository hoặc toàn bộ | `hammer capsule content verify-checksum --id <CAPSULE_ID> [--lifecycle-environment-id <LE_ID> \| --content-view-id <CV_ID> \| --repository-id <REPO_ID>] [--organization-id <ORG_ID>]` |
| Metadata publish không khớp nội dung repository | Republish Repository Metadata (Web UI: Select Action); không áp dụng cho repository Complete Mirroring | tài liệu 6.19 ghi `hammer repository verify-checksum --name "<REPO_NAME>" --product "<PRODUCT>" --organization-id <ORG_ID>` |
| Host báo checksum mismatch từ một CV version | Republish metadata của CV version (Versions → options) | `hammer content-view version verify-checksum --id <CVV_ID>` |

Red Hat khuyến nghị ưu tiên Complete Sync hoặc publish CV version mới thay cho republish metadata. Tên lệnh republish thay đổi giữa các bản: kiểm tra `hammer repository --help` trên bản đang chạy. Theo dõi kết quả bằng `hammer task list --search "Verify checksum for content"`.

## Lỗi hay gặp (kinh nghiệm thực tế)

- Sync bị ngắt (Satellite thiếu RAM, mất mạng) rồi chạy lại báo Success nhưng repository có 0 gói: chạy lại với `--skip-metadata-check true`.
- Host dùng CV chứa repository minor (ví dụ RHEL 10.1) mà chưa khóa release thì `$releasever` là bản major, dnf báo 404: trên host chạy `subscription-manager release --set=<MINOR>`, `subscription-manager refresh`, `dnf clean all`.
- Host đăng ký xong không thấy repository Satellite Client/Capsule: các repository này mặc định Disabled trong Activation Key, cần override Enabled (`hammer activation-key product-content --name "<ACTIVATION_KEY>" --organization "<ORG>"` để kiểm tra).
- RHEL 10 báo GPG check FAILED: xem [khóa release 4 của RHEL 10](../../rhel/rhel10-gpg-release-key-4.md).

## Liên quan

- [Download policy của repository](download-policy.md)
- [Alternate content source và mirroring policy](alternate-content-sources.md)
- [Content View và Lifecycle Environment](content-views-and-lifecycle.md)
- [Subscription manifest và SCA](subscriptions-manifest.md)
- [Các loại nội dung khác: container, Flatpak, ISO, Ansible, file](other-content-types.md)
- [Cổng mạng](../architecture/network-ports.md) · [Quản lý dung lượng](../operations/storage-management.md)
- [Activation Key chỉ áp dụng lúc đăng ký](../hosts/activation-key-registration-only.md)

## Nguồn

- Red Hat Satellite 6.19, Managing content, chương 3 Adding content to Satellite (PDF tr. 20–31) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing content, chương 4 Synchronizing content to Satellite (PDF tr. 32–41) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing content, chương 10 Recovering corrupted content (PDF tr. 101–106) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing content, phụ lục A Required Red Hat repositories (PDF tr. 172–173) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- RH403 (Satellite 6.15), chương 2 Manage software lifecycles, chương 5 Deploy custom software: giải thích product, GPG key, sync plan.
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: danh sách repository dùng thật, sync plan hằng ngày, xử lý sync báo 0 gói, lỗi 404 do chưa khóa release, override repository trong Activation Key.
