---
title: Alternate content source và tối ưu đồng bộ, lưu trữ (mirroring policy)
tags: [howto, satellite, capsule, pulp, performance, hammer]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Managing content, chương 5, mục 6.6–6.13, phụ lục B (PDF tr. 42–49, 52–56, 174–175); Administering, phụ lục F.7 (PDF tr. 140–143)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Alternate content source (ACS) cho Satellite/Capsule tải gói từ một nguồn gần hơn (thư mục cục bộ, mirror nội bộ, RHUI hoặc thẳng Red Hat CDN) trong khi metadata vẫn lấy từ upstream; tạo xong phải Refresh. Muốn giảm dung lượng và băng thông thì kết hợp mirroring policy, giới hạn số phiên bản gói giữ lại và download concurrency.

## ACS là gì

- Khi sync, **metadata** vẫn lấy từ upstream URL hoặc từ Satellite, còn **gói** được tải từ ACS nếu ACS có. Hữu ích khi nội dung đã nằm trên đĩa cục bộ/NFS, trên mirror trong mạng LAN, hoặc khi Capsule có đường ra CDN nhanh hơn đường về Satellite.
- Áp dụng cho Satellite Server và Capsule Server (chọn Capsule khi tạo ACS).
- Phải **Refresh** ACS sau khi tạo hoặc sửa; một cron hằng tuần tự refresh mọi ACS.
- ACS gắn với Satellite Server hoặc với Capsule phục vụ nhiều Organization có tác dụng lên **mọi Organization**.

| Loại | Lấy gói từ đâu | Dùng khi |
|---|---|---|
| Custom | Bất kỳ repository nào trên mạng hoặc filesystem (base URL + danh sách subpath) | Có mirror nội bộ, thư mục cục bộ |
| Simplified | Chép thông tin upstream của các product đã chọn từ Satellite; với product Red Hat, Capsule tải thẳng từ Red Hat CDN | Capsule ra Internet/upstream nhanh hơn về Satellite |
| RHUI | Red Hat Update Infrastructure (RHUI 4 trở lên, cấu hình mặc định; AWS RHUI không hỗ trợ) | Đã có hạ tầng RHUI |

Subpath được ghép vào base URL: base `https://<MIRROR_FQDN>` và subpath `rhel9/,rhel8/` → Satellite tìm ở `https://<MIRROR_FQDN>/rhel9/` và `https://<MIRROR_FQDN>/rhel8/`.

## Tạo ACS

Web UI: **Content → Alternate Content Sources → Add source** → chọn Source type (Custom, Simplified, RHUI), Content type, tên, Capsule áp dụng, tùy chọn **Use HTTP proxies** (dùng proxy của Capsule), base URL và subpath hoặc product, xác thực, Verify SSL → **Add** → menu tùy chọn của ACS → **Refresh**.

```bash
# Custom ACS (mirror nội bộ)
hammer alternate-content-source create \
  --alternate-content-source-type custom \
  --base-url "https://<MIRROR_FQDN>:<PORT>" \
  --name "<ACS_NAME>" \
  --smart-proxy-ids <CAPSULE_ID_1>,<CAPSULE_ID_2> \
  --subpaths "rhel9/,rhel8/" \
  --verify-ssl true

# Simplified ACS theo product (yum; dùng --content-type file cho product chứa repository file)
hammer alternate-content-source create \
  --alternate-content-source-type simplified \
  --content-type yum \
  --name "<ACS_NAME>" \
  --product-ids <PRODUCT_ID_1>,<PRODUCT_ID_2> \
  --smart-proxy-ids <CAPSULE_ID_1>

# RHUI ACS: subpath lấy từ "rhui-manager repo info --repo_id <REPO_ID>" trên RHUA
hammer alternate-content-source create \
  --alternate-content-source-type rhui \
  --base-url "https://<RHUI_CDS_FQDN>/pulp/content" \
  --name "<ACS_NAME>" \
  --smart-proxy-ids <CAPSULE_ID_1> \
  --ssl-client-cert-id <SSL_CERT_ID> --ssl-client-key-id <SSL_KEY_ID> \
  --subpaths path/to/repo/1/,path/to/repo/2/ \
  --verify-ssl true

hammer alternate-content-source refresh --name "<ACS_NAME>"
hammer task list --search "Refresh Alternate Content Source"
```

- Upstream cần SSL/chứng chỉ client: import vào Content Credentials trước (xem [repository và đồng bộ](repositories-and-sync.md)).
- **Capsule sync thẳng từ Red Hat CDN**: tạo Simplified ACS, Content type Yum, chọn Capsule và các product Red Hat, rồi Refresh. Giảm tải cho Satellite nhưng Capsule phải ra được CDN (trực tiếp hoặc qua proxy).
- Quyền: xem ACS cần `view_content_credentials`, `view_organizations`, `view_products`, `view_smart_proxies`; quản lý cần thêm `create_/edit_/destroy_/view_alternate_content_sources`.

## Mirroring policy

Mirroring policy quyết định sync có xóa nội dung mà upstream đã xóa hay không:

| Policy | Giá trị hammer | Hành vi |
|---|---|---|
| Additive | `additive` | Chỉ thêm nội dung mới, không xóa gì, không mirror repodata |
| Content Only | `mirror_content_only` | Mirror nội dung (xóa gói upstream đã xóa), tự sinh repodata; dùng khi upstream không hỗ trợ mirror metadata |
| Complete Mirroring | `mirror_complete` | Mirror cả nội dung lẫn repodata; nhanh nhất; chỉ cho yum |

- Repository Red Hat mặc định `mirror_complete` và không chịu ảnh hưởng của setting mặc định.
- Tránh republish metadata cho repository (và content view chứa repository) dùng Complete Mirroring — metadata được chép nguyên từ upstream.

```bash
# Mặc định cho repository custom tạo mới (không đổi repository đã có)
hammer settings set --name default_yum_mirroring_policy --value mirror_complete
hammer settings set --name default_non_yum_mirroring_policy --value additive
# Đổi cho một repository
hammer repository update --id <REPO_ID> --mirroring-policy mirror_content_only
```

## Giới hạn tăng trưởng và tải đồng bộ

- **Giữ ít phiên bản gói** (chỉ với Additive): Retain package versions = 1 giữ bản mới nhất, ≥ 2 giữ nhiều bản.

```bash
hammer repository update --id <REPO_ID> \
  --mirroring-policy additive \
  --retain-package-versions-count <N>
```

- **Download concurrency** theo repository: giảm khi upstream từ chối request song song, tăng thận trọng để sync nhanh hơn.

```bash
hammer repository update --id <REPO_ID> --download-concurrency 5 --organization "<ORG>"
```

- Setting liên quan (Administer → Settings → Content): **Sync Connection Timeout** (mặc định 300 giây), **Batch size to sync repositories in** (mặc định 100 repository đồng thời trên một Capsule), **Sync Capsules after Content View promotion** (mặc định Yes).
- Download policy (`immediate`/`on_demand`, Capsule Inherit/Streamed) là đòn bẩy lớn nhất về dung lượng: xem trang [download policy](download-policy.md), không nhắc lại ở đây.
- Content view version cũ cũng chiếm chỗ: purge bằng `hammer content-view purge` ([Content View và Lifecycle Environment](content-views-and-lifecycle.md)).
- Đĩa cục bộ không đủ: có thể mount NFS vào `/var/lib/pulp` (phụ lục B: dừng dịch vụ, chép dữ liệu, mount với SELinux context `pulpcore_var_lib_t`); Red Hat lưu ý Pulp cần lưu trữ độ trễ thấp, băng thông cao. Quy trình chi tiết: [quản lý dung lượng](../operations/storage-management.md).

## Lưu ý thiết kế

- Simplified ACS với product Red Hat chỉ hợp khi chính sách bảo mật cho phép Capsule ra Internet. Ở mô hình Capsule nằm trong vùng nội bộ không có đường ra CDN (mô hình thường gặp ở môi trường kiểm soát mạng chặt), chỉ Satellite đi CDN; khi đó dùng Custom ACS trỏ về mirror nội bộ hoặc không dùng ACS.
- ACS không thay thế Inter-Satellite Sync: ACS chỉ đổi nơi tải gói, metadata vẫn cần upstream. Satellite không có đường ra ngoài thì xem [đồng bộ giữa các Satellite](inter-satellite-sync.md).
- Trang này tổng hợp từ tài liệu hãng; ACS chưa được dùng trong các triển khai thực tế đã tham chiếu.

## Liên quan

- [Download policy của repository](download-policy.md)
- [Repository Red Hat, custom repository và đồng bộ](repositories-and-sync.md)
- [Đồng bộ nội dung giữa các Satellite](inter-satellite-sync.md)
- [Cài Capsule](../installation/install-capsule.md) · [Sizing và tuning](../architecture/sizing-and-tuning.md)
- [Quản lý dung lượng](../operations/storage-management.md)

## Nguồn

- Red Hat Satellite 6.19, Managing content, chương 5 Managing alternate content sources (PDF tr. 42–49) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing content, mục 6.6–6.13 Mirroring policies, concurrency, retain versions (PDF tr. 52–56) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing content, phụ lục B Storing content on an NFS share (PDF tr. 174–175) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Administering Red Hat Satellite, phụ lục F.7 Content management settings (PDF tr. 140–143) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
