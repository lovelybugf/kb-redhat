---
title: Errata trong Content View, incremental update và quy trình vá khẩn
tags: [howto, satellite, errata, content-view, hammer]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Managing content, chương 13, mục 9.3–9.4 (PDF tr. 138–146, 78–84); Administering, phụ lục F.7 (PDF tr. 140–143); RH403 chương 4; kinh nghiệm triển khai thực tế"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Errata mới sync chỉ nằm trong Library nên trên host là Applicable; muốn Installable phải publish version mới hoặc incremental update rồi promote tới LE của host và chờ Capsule đồng bộ. Vá khẩn một RHSA dùng `hammer content-view version incremental-update` trên version đang ở LE, kiểm tra Capsule, rồi vá host bằng `dnf update-minimal --advisory`.

## Loại errata và trạng thái

| Loại | Mã | Nội dung |
|---|---|---|
| Security Advisory | RHSA | Sửa lỗ hổng, có CVE, mức Critical/Important/Moderate/Low; có thể chứa cả bugfix và enhancement |
| Bug Fix Advisory | RHBA | Sửa lỗi; có thể chứa enhancement |
| Product Enhancement Advisory | RHEA | Chỉ tính năng mới |

| Trạng thái trên host | Nghĩa | Việc cần làm |
|---|---|---|
| Applicable | Host đang cài gói bị ảnh hưởng, bản vá có trong Library nhưng **chưa** có trong CV version ở LE của host | Publish/promote hoặc incremental update |
| Installable | Bản vá đã có trong content view environment của host, chưa cài | Vá trên host (dnf hoặc job từ Satellite) |

Installable luôn là Applicable. Setting **Installable errata from Content View** (mặc định No): khi bật, trạng thái errata của host chỉ tính theo errata có trong CV/LE của host. Satellite nhập metadata errata mỗi lần sync repository từ CDN.

## Xem errata

Web UI: **Content → Content Types → Errata**; mặc định tick **Applicable**, tick thêm **Installable** để chỉ xem cái cài được. Trang erratum có tab Details, Content Hosts (cài cho host), Repositories (repository/CV/LE đã chứa erratum).

```bash
hammer erratum list --organization "<ORG>" --search "type = security and severity = Critical"
hammer erratum info --id <ERRATUM_ID>
hammer erratum list --product-id <PRODUCT_ID> --errata-restrict-applicable 1 --order "type desc" \
  --search "cve = CVE-YYYY-NNNNN"
hammer host errata list --host "<HOST_FQDN>"
```

| Tham số tìm kiếm | Ví dụ |
|---|---|
| `type` | `type = security` (security, bugfix, enhancement) |
| `severity` | `severity = Important` |
| `cve`, `bug`, `id` | `cve = CVE-YYYY-NNNNN`, `id = RHSA-YYYY:NNNN` |
| `package_name`, `package` | `package_name = kernel` |
| `issued`, `updated` | `issued > "7 days ago"`, `updated < "Jan 12,2026"` |
| `title` | `title ~ openssl` |

Kết hợp bằng `and`/`or`. Thông báo email (Administer → Users → Email Preferences): tóm tắt errata của host, errata mới sau sync, errata của CV vừa promote.

## Filter errata theo ngày: đóng băng nhưng vẫn nhận bản vá bảo mật

Mẫu phổ biến cho PROD: Exclude errata Bugfix và Enhancement từ một ngày chốt, giữ mọi Security.

```bash
hammer content-view filter create --content-view "<CONTENT_VIEW>" --organization "<ORG>" \
  --name "freeze-nonsec-<YYYY-MM>" --type erratum --inclusion false \
  --description "Exclude bugfix/enhancement updated from <YYYY-MM-DD>"
hammer content-view filter rule create --content-view "<CONTENT_VIEW>" --organization "<ORG>" \
  --content-view-filter "freeze-nonsec-<YYYY-MM>" \
  --start-date "<YYYY-MM-DD>" --date-type updated --types bugfix,enhancement
hammer content-view publish --name "<CONTENT_VIEW>" --organization "<ORG>" --description "Errata freeze <YYYY-MM-DD>"
hammer content-view version promote --content-view "<CONTENT_VIEW>" --version <VERSION> \
  --to-lifecycle-environment "<LIFECYCLE_ENV>" --organization "<ORG>"
```

- Web UI: CV → Filters → Create filter, Content type **Errata - by date range**, **Exclude**; tick Bugfix + Enhancement, bỏ Security; Date type Updated (hoặc Issued); Start date = ngày chốt, End date để trống.
- Muốn loại cả Security sau ngày chốt thì thêm `security` vào `--types`.
- Filter chỉ áp dụng cho version publish sau đó; errata bị loại vẫn hiện Applicable trên host. Đổi ngày chốt theo chu kỳ (tháng/quý) rồi publish lại.
- Không bật Solve dependencies cho CV có filter bảo mật (dependency được ưu tiên hơn luật filter).

## Incremental update

Thêm một hoặc vài errata (và gói phụ thuộc) vào một CV version đang có, tạo version minor (ví dụ 3.0 → 3.1) mà không mang toàn bộ nội dung mới của Library.

Web UI: **Content → Content Types → Errata** → chọn erratum → **Apply to Hosts** (hoặc tab Content Hosts) → chọn host → trang Confirm hiển thị CV, version mới, LE, số host → **Confirm**. Không tick "Apply Errata to Content Hosts immediately after publishing" nếu chưa bật Remote Execution.

```bash
hammer content-view version list --content-view "<CONTENT_VIEW>" --organization "<ORG>"
hammer content-view version incremental-update \
  --content-view-version-id <CVV_ID> \
  --errata-ids <ERRATUM_ID_1>,<ERRATUM_ID_2> \
  --lifecycle-environment-ids <LE_ID>
```

- Lệnh in danh sách errata và gói được thêm; mặc định Satellite tự thêm gói phụ thuộc (một errata thư viện có thể kéo theo hàng chục gói) — bình thường, giúp dnf không thiếu dependency.
- Description của version minor tự ghi errata đã thêm; dùng Compare với version gốc để xem chính xác thay đổi.
- Version minor thay cho version cũ **ở LE đó**, áp dụng cho **mọi host** trong LE dùng CV này, không riêng host đã chọn.

### Giới hạn

- CV version càng cũ (cách xa bản minor RHEL hiện tại) thì càng dễ lỗi dependency: Satellite kéo bản mới nhất của gói phụ thuộc, có thể không cài chung được với gói cũ còn trong repository; tài liệu nêu hiện chưa có cách khắc phục. Khi dnf báo lỗi dependency, publish version đầy đủ thay vì incremental.
- Thêm errata Enhancement vào version cũ dễ không tương thích; incremental update nên dành cho RHSA.

## Quy trình vá khẩn (RHSA Critical/Important)

1. Xác định phạm vi: `hammer erratum list --search "cve = <CVE>"` (mỗi bản RHEL có RHSA riêng); trên host kiểm tra `dnf updateinfo list --cve <CVE>` — rỗng nghĩa là chưa Installable.
2. Lấy ID version đang ở LE đích (thường DEV trước): `hammer content-view version list`.
3. Incremental update errata vào version đó cho LE DEV (lệnh ở trên); theo dõi task ở Monitor → Satellite Tasks.
4. Kiểm tra mọi Capsule phục vụ LE đã đồng bộ version minor (Infrastructure → Capsules → tab Content).
5. Trên host: `subscription-manager refresh; dnf clean all; dnf update-minimal --advisory <ERRATUM_ID> -y`; `dnf needs-restarting -r` rồi reboot trong khung giờ nếu là kernel/glibc/systemd; `dnf uploadprofile --force-upload` để Satellite cập nhật trạng thái.
6. Kiểm thử, phê duyệt thay đổi, rồi incremental update (hoặc promote version minor) lên PROD và lặp bước 4–5.
7. Hoàn tác: promote lại version cũ vào LE; trên host `dnf history undo last`.

Lưu ý phạm vi: sau bước 3 mọi host trong LE đều thấy bản vá; job vá tự động (dnf-automatic, cron, Ansible) trên các host đó sẽ cài luôn. Ghi rõ phạm vi vào phiếu thay đổi. Đợt publish định kỳ sau sẽ thay version minor, môi trường đồng nhất lại.

Vá trên host (Remote Execution, vá hàng loạt, báo cáo trước/sau): xem [vá host bằng errata](../hosts/patching-with-errata.md) và [report template, cảnh báo CVE](../hosts/report-templates-and-cve-alert.md).

## Chu kỳ vá mẫu (đề xuất đã dùng trong dự án, ẩn danh)

Red Hat không quy định chu kỳ; mô hình tài liệu 6.19 là: nội dung đóng băng theo CV version, đi DEV rồi PROD, RHSA khẩn qua incremental update, tự động hóa bằng Hammer/Ansible.

| Mức | Thời hạn đề xuất | Đường đi nội dung |
|---|---|---|
| RHSA Critical | DEV trong 48 giờ, PROD trong 7 ngày | Incremental update, DEV → PROD |
| RHSA Important | PROD trong 30 ngày | Gộp vào đợt tháng; incremental nếu CVE đang bị khai thác |
| RHSA Moderate/Low, RHBA, RHEA | Đợt tháng hoặc quý | Publish version mới đầu kỳ |

Lịch tháng: ngày đầu tháng publish và promote DEV; tuần 1–2 vá DEV, kiểm thử ứng dụng; tuần 3 promote PROD và vá theo nhóm host; tuần 4 xuất báo cáo errata cho bộ phận an ninh. Chỉ số nên theo dõi: số host còn RHSA Critical Installable, tuổi CV version ở PROD (ví dụ không quá 45 ngày).

## Kinh nghiệm thực tế

- Đã kiểm chứng trên lab và môi trường UAT (ẩn danh): incremental update tạo version 2.0 → 2.1 chỉ ở LE đích, host chỉ thấy đúng errata vừa thêm, các errata khác vẫn ẩn; vá bằng `dnf update-minimal --advisory` thành công.
- "Satellite đã có bản vá mà host chưa thấy" gần như luôn do bản vá mới ở Library: publish/promote (hoặc incremental) rồi chờ Capsule sync.
- Promote hay incremental update không tự cài gì lên host; bằng chứng kiểm toán là báo cáo errata trước/sau và `dnf history info` trên host.

## Liên quan

- [Content View và Lifecycle Environment](content-views-and-lifecycle.md)
- [Vá host bằng errata](../hosts/patching-with-errata.md)
- [Report template và cảnh báo CVE](../hosts/report-templates-and-cve-alert.md)
- [Remote Execution](../hosts/remote-execution.md)
- [Repository và đồng bộ](repositories-and-sync.md)

## Nguồn

- Red Hat Satellite 6.19, Managing content, chương 13 Making errata available through content views (PDF tr. 138–146) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing content, mục 9.3–9.4 Content view filters, dependency solving (PDF tr. 78–84) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Administering Red Hat Satellite, phụ lục F.7 Content management settings (PDF tr. 140–143) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- RH403 (Satellite 6.15), chương 4 Manage and apply errata to hosts: Applicable/Installable, khuyến nghị dùng version minor.
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: incremental update trên lab và UAT, quy trình vá khẩn, chính sách vá hằng tháng đề xuất cho khách hàng.
