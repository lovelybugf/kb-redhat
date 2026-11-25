---
title: Đồng bộ nội dung giữa các Satellite (ISS) bằng mạng hoặc export/import
tags: [howto, satellite, hammer, pulp, installation]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Managing content, chương 11 (PDF tr. 107–129); Installing Satellite Server in a disconnected network environment, mục 5.2–5.4, phụ lục B–C (PDF tr. 30–35, 48–49); Overview, concepts, and deployment considerations, mục 12.6 (PDF tr. 57–59)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Satellite disconnected nhận nội dung từ Satellite upstream có Internet bằng Network Sync (kéo qua HTTPS) hoặc Export Sync cho máy air-gapped (export, chép đĩa, import). Cấu hình bằng `hammer organization configure-cdn`, chuyển nội dung bằng `hammer content-export`/`content-import`; upstream phải để repository ở `immediate`.

## Chọn kịch bản

| Kịch bản | Kết nối upstream ↔ downstream | Cấu hình CDN ở downstream | Cách lấy nội dung |
|---|---|---|---|
| Disconnected (ISS Network Sync) | Có mạng nội bộ, downstream không ra Internet | `network_sync` | Sync repository như bình thường, nguồn là upstream |
| Air-gapped (ISS Export Sync) | Không có mạng | `export_sync` | Export → mang đĩa → import |
| Custom CDN | Có web server nội bộ mirror cấu trúc Red Hat CDN (hoặc chứa export syncable) | `custom_cdn` | Sync từ web server |

- ISS dùng giữa **Satellite với Satellite**; Capsule đã tự đồng bộ từ Satellite, không dùng ISS.
- Có thể đồng bộ toàn bộ Library, chỉ một số CV version (upstream curate nội dung), hoặc từng repository.
- Export/import dạng mặc định (importable) đòi hỏi upstream và downstream **cùng major version** Satellite. Không khớp version thì dùng định dạng syncable hoặc Network Sync.

## Điều kiện chung

- Upstream: repository cần đồng bộ để **Immediate** (đổi repository đang On Demand; đặt `default_redhat_download_policy` và `default_download_policy` là `immediate` trước khi bật repository mới — xem [download policy](download-policy.md)). Sync upstream đến thời điểm cần.
- Upstream đủ chỗ trống ở thư mục export và ở `/var/lib/pulp/exports/` (tương đương dung lượng repository) cho file tạm.
- Người export có role **Content Exporter**; người import có role **Content Importer**.
- Downstream: manifest của Organization phải chứa subscription cho các sản phẩm Red Hat có trong nội dung nhập; với Network Sync dùng **cùng manifest** như upstream.

## Network Sync

Điều kiện thêm: user ở upstream là admin hoặc có `view_organizations`, `view_products`, `export_content`, `view_lifecycle_environments`, `view_content_views`; downstream đã import CA của upstream (`http://<UPSTREAM_FQDN>/pub/katello-server-ca.crt`) thành content credential loại SSL.

Web UI downstream: **Content → Subscriptions → Manage Manifest → CDN Configuration → Network Sync** → URL upstream, username, password hoặc personal access token, Organization label upstream, (tùy chọn) LE label (mặc định Library) và CV label (mặc định Default_Organization_View), SSL CA → **Update**; sau đó bật repository và Sync Now hoặc gắn sync plan.

```bash
hammer content-credential show --name "<UPSTREAM_CA_NAME>" --organization "<ORG>"
hammer organization configure-cdn \
  --name "<ORG>" \
  --type network_sync \
  --url https://<UPSTREAM_FQDN> \
  --username <USER> --password <PASSWORD> \
  --ssl-ca-credential-id <CA_CREDENTIAL_ID> \
  --upstream-organization-label "<UPSTREAM_ORG_LABEL>" \
  --upstream-lifecycle-environment-label "<LIFECYCLE_ENV>" \
  --upstream-content-view-label "<CONTENT_VIEW>"
```

Repository **custom** không tự đi theo Network Sync: ở upstream copy **Published At** URL của repository; ở downstream tạo product/repository custom với Upstream URL đó và SSL CA của upstream.

## Export ở upstream

```bash
# Toàn bộ Library của Organization (tạo CV ẩn Export-Library)
hammer content-export complete library --organization "<ORG>"
# Lần sau chỉ phần thay đổi
hammer content-export incremental library --organization "<ORG>"

# Một CV version (đã curate)
hammer content-export complete version --content-view "<CONTENT_VIEW>" --version 1.0 --organization "<ORG>"
hammer content-export incremental version --content-view "<CONTENT_VIEW>" --version 2.0 --organization "<ORG>"
# Version đang ở một LE
hammer content-export complete version --content-view "<CONTENT_VIEW>" --lifecycle-environment "<LIFECYCLE_ENV>" --organization "<ORG>"

# Một repository
hammer content-export complete repository --name "<REPO_NAME>" --product "<PRODUCT>" --organization "<ORG>"
hammer content-export incremental repository --name "<REPO_NAME>" --product "<PRODUCT>" --organization "<ORG>"

# Theo dõi
hammer content-export list --organization "<ORG>"
```

Tùy chọn dùng chung:

| Tùy chọn | Tác dụng |
|---|---|
| `--chunk-size-gb=2` | Chia archive thành khối 2 GB (export lớn thường vài GB) |
| `--format=syncable` | Xuất dạng cây thư mục yum có thể phục vụ qua HTTP hoặc import; chỉ hỗ trợ yum, kickstart, file (không Ansible collection, không docker) |
| `--destination-server=<NAME>` | Ghi nhận export dành cho downstream nào, để quản lý chuỗi incremental |

- Export mặc định nằm ở `/var/lib/pulp/exports/<ORG>/<CV_or_Export-...>/<VERSION>/<TIMESTAMP>/`; cần **đủ ba loại file**: `*.tar.gz` (có thể nhiều khối), `*-toc.json`, `metadata.json`.
- Nội dung export được: yum, kickstart, file, docker, Ansible collection (dạng importable).
- Không export định nghĩa CV và filter; CCV xuất ra thành CV thường.
- Có nội dung mới cần đưa xuống: republish CV ở upstream trước, rồi mới export incremental.

## Import ở downstream

Cấu hình Organization nhận nội dung bằng export (Web UI: CDN Configuration → **Export Sync**):

```bash
hammer organization configure-cdn --name "<ORG>" --type export_sync
```

Chạy bằng root trên downstream:

```bash
mkdir -p /var/lib/pulp/imports/<EXPORT_DIR>
# chép toàn bộ file export vào thư mục trên
chown -R pulp:pulp /var/lib/pulp/imports/<EXPORT_DIR>
ls -lh /var/lib/pulp/imports/<EXPORT_DIR>
```

```bash
# Đường dẫn phải tuyệt đối, đường dẫn tương đối không chạy
hammer content-import library    --organization "<ORG>" --path /var/lib/pulp/imports/<EXPORT_DIR>/
hammer content-import version    --organization "<ORG>" --path /var/lib/pulp/imports/<EXPORT_DIR>/
hammer content-import repository --organization "<ORG>" --path /var/lib/pulp/imports/<EXPORT_DIR>/
# Export syncable phục vụ qua web server
hammer content-import library --organization "<ORG>" --path http://<WEB_SERVER>/pub/exports/<EXPORT_DIR>/
hammer content-view version list --organization-id <ORG_ID>
```

- Import Library tạo CV ẩn **Import-Library** chỉ để phục vụ import; không gán host vào CV này, dùng Default Organization View hoặc CV riêng như bình thường.
- Import CV version giữ nguyên số major.minor, cùng repository, gói, errata; product/repository/CV custom tự tạo nếu chưa có.
- Thư mục import phải nằm trong `ALLOWED_IMPORT_PATHS` (`/etc/pulp/settings.py`, mặc định gồm `/var/lib/pulp/imports/`).
- Import thành công thì xóa thư mục import để giải phóng đĩa: `rm -rf /var/lib/pulp/imports/<EXPORT_DIR>/`.

## Custom CDN và quay lại Red Hat CDN

```bash
hammer organization configure-cdn --name "<ORG>" --type custom_cdn \
  --url https://<CDN_FQDN> --ssl-ca-credential-id <CA_CREDENTIAL_ID>
# Môi trường chuyển sang connected
hammer organization configure-cdn --name "<ORG>" --type redhat_cdn
```

Export syncable có thể đặt lên web server nội bộ rồi trỏ Custom CDN vào đó; Red Hat khuyến nghị import trực tiếp bằng `hammer content-import` khi hai bên cùng phiên bản.

## Lưu ý

- Lỗi "Path ... is not an allowed export path" khi export: kiểm tra thư mục đích (Red Hat có bài Knowledgebase riêng cho lỗi này).
- Lên lịch: export incremental đều đặn (ví dụ hằng tuần) giữ gói chuyển nhỏ; lưu sổ export theo `--destination-server` để không lệch chuỗi incremental.
- Quy trình vận hành của downstream (CV, LE, errata) giữ nguyên như Satellite connected sau khi nội dung đã vào Library.
- Trang này tổng hợp từ tài liệu hãng; ISS chưa nằm trong phạm vi các triển khai thực tế đã tham chiếu (các Satellite đó đều đi CDN trực tiếp hoặc qua proxy), nên các lệnh chưa được chạy thử.

## Liên quan

- [Cài Satellite disconnected](../installation/install-satellite-disconnected.md)
- [Subscription manifest và cấu hình CDN](subscriptions-manifest.md)
- [Download policy của repository](download-policy.md)
- [Content View và Lifecycle Environment](content-views-and-lifecycle.md)
- [Quản lý dung lượng](../operations/storage-management.md)

## Nguồn

- Red Hat Satellite 6.19, Managing content, chương 11 Synchronizing content between Satellite Servers (PDF tr. 107–129) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Installing Satellite Server in a disconnected network environment, mục 5.2–5.4 (PDF tr. 30–35), phụ lục B–C (PDF tr. 48–49) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Overview, concepts, and deployment considerations, mục 12.6 Inter-Satellite Synchronization scenarios (PDF tr. 57–59) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
