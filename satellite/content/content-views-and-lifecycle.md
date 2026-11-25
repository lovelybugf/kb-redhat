---
title: Content View, Lifecycle Environment và content view environment
tags: [concept, satellite, content-view, capsule, hammer]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Managing content, chương 7, 8, 9, mục 12.8 (PDF tr. 57–100, 134–137); Overview, concepts, and deployment considerations, mục 12.3 (PDF tr. 53–54); RH403 chương 2, 4; kinh nghiệm triển khai thực tế"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Content View (CV) là snapshot có phiên bản của repository trong Library; publish tạo version mới, promote đưa version qua từng Lifecycle Environment (LE) như Library → DEV → PROD. Host chỉ thấy gói của version đang ở LE của nó; từ 6.16 host gắn được nhiều content view environment theo thứ tự ưu tiên, thay cho Composite CV.

## Mô hình

| Khái niệm | Ý nghĩa |
|---|---|
| Library | LE gốc, luôn chứa nội dung sync mới nhất; chứa **Default Organization View** (mọi repository đã sync) |
| Lifecycle Environment | Một giai đoạn (DEV, TEST, PROD...); chuỗi LE nối từ Library gọi là environment path |
| Content View | Tập repository đã chọn (+ filter); mỗi lần **publish** tạo version bất biến (1.0, 2.0...) |
| Promote | Gán một CV version vào LE kế tiếp; mỗi LE chỉ giữ một version của mỗi CV |
| Incremental update | Thêm errata vào version đang có, tăng số minor (2.0 → 2.1); xem [errata và incremental update](errata-and-incremental-update.md) |
| Composite CV (CCV) | Ghép version của nhiều CV thành một |
| Rolling CV | CV không publish/promote, luôn theo nội dung mới nhất của Library cho các LE được gán |
| Content view environment | Cặp `<LE_label>/<CV_label>` gán cho host/Activation Key; CV mới publish chưa promote có dạng `Library/<CV_label>` |

Promote **không cài gì** lên host: nó chỉ làm gói sẵn sàng cho mọi host trong LE; host phải chạy dnf hoặc nhận job từ Satellite. Gặp lỗi sau khi vá thì gán host (hoặc promote lại) version cũ — chỉ đổi nguồn gói, không hạ phiên bản gói đã cài.

## Thiết kế Lifecycle Environment

- Không đăng ký host vào Library (không có bước kiểm thử) và **không gán Library cho Capsule** (mỗi lần CDN cập nhật là Capsule sync lại, tốn tài nguyên, băng thông, đĩa).
- Mô hình mặc định: các giai đoạn cố định DEV → TEST → PROD, publish định kỳ và promote dần sau kiểm thử.
- Mô hình thay thế: mỗi LE là một mốc nội dung cố định (ví dụ 2026-Q4, 2027-Q1), errata khẩn đưa vào bằng incremental update; khi vá thì chuyển host sang LE mới. Ưu điểm: nhìn LE là biết host ở mức vá nào; nhược điểm: kém linh hoạt.
- Nhiều environment path khi các nhóm host có nhịp phát hành khác nhau (ví dụ OS chỉ cần TEST → PROD, ứng dụng cần QA và UAT). Capsule đồng bộ theo LE nên tách path hợp lý giúp Capsule chỉ tải nội dung cần.

```bash
hammer lifecycle-environment create --name "DEV" --prior "Library" --organization "<ORG>"
hammer lifecycle-environment create --name "PROD" --prior "DEV" --organization "<ORG>"
hammer lifecycle-environment paths --organization "<ORG>"
# Gán LE cho Capsule rồi đồng bộ
hammer capsule content available-lifecycle-environments --id <CAPSULE_ID>
hammer capsule content add-lifecycle-environment --id <CAPSULE_ID> \
  --lifecycle-environment-id <LE_ID> --organization "<ORG>"
hammer capsule content synchronize --id <CAPSULE_ID> [--lifecycle-environment-id <LE_ID>]
# Gỡ LE khỏi Capsule
hammer capsule content remove-lifecycle-environment --id <CAPSULE_ID> --lifecycle-environment-id <LE_ID>
```

Chỉ xóa được LE khi không còn host/AK/CV version dùng nó (`hammer lifecycle-environment delete --name "<LIFECYCLE_ENV>" --organization "<ORG>"`).

## Thiết kế Content View

- CV gom cả OS và công cụ đi kèm dễ bảo trì hơn nhiều CV quá nhỏ; tránh tạo quá nhiều CV vì kích thước và số lượng CV làm chậm publish/promote.
- Kịch bản thường gặp (Overview 12.3): một CV cho tất cả; một CV cho mỗi loại host; Composite CV cho mỗi loại host (OS và ứng dụng tách nhịp cập nhật); CV theo thành phần (ví dụ database) dùng lại trong nhiều CCV.
- Không đưa Kickstart repository vào CV (chỉ dùng cho provisioning).
- Ghi ngày và nội dung thay đổi vào **Description** của mỗi version; Web UI chỉ hiển thị ngày hoạt động gần nhất (kể cả promote), không phải ngày nội dung.
- Số major tăng mỗi lần publish, minor tăng khi incremental update; không reset được bộ đếm. CV sao chép (Copy) bắt đầu từ version cuối của CV gốc, không có lịch sử cũ.
- Đặt quy ước tên ngay từ đầu, ví dụ `cv-<os|ứng dụng>` và `ccv-<ứng dụng>-<os>`, chỉ chữ thường, số, gạch ngang.

## Tạo, publish, promote

```bash
hammer repository list --organization "<ORG>"
hammer content-view create --name "<CONTENT_VIEW>" --organization "<ORG>" \
  --repository-ids <REPO_ID_1>,<REPO_ID_2>
hammer content-view add-repository --name "<CONTENT_VIEW>" --organization "<ORG>" --repository-id <REPO_ID_3>
hammer content-view publish --name "<CONTENT_VIEW>" --organization "<ORG>" \
  --description "2026-11: sync ngay <YYYY-MM-DD>, them repo X"
hammer content-view version list --content-view "<CONTENT_VIEW>" --organization "<ORG>"
hammer content-view version promote --content-view "<CONTENT_VIEW>" --version 3.0 \
  --to-lifecycle-environment "DEV" --organization "<ORG>"
hammer content-view version info --id <CVV_ID>
# Đồng bộ riêng một CV xuống Capsule (Web UI chỉ sync theo cả các LE)
hammer capsule content synchronize --id <CAPSULE_ID> --content-view "<CONTENT_VIEW>"
```

- Quyền cần: `promote_or_remove_content_views` và `promote_or_remove_content_views_to_environment`.
- Setting **Sync Capsules after Content View promotion** mặc định Yes: promote xong Capsule có LE đó tự đồng bộ.
- Promote nên đi tuần tự qua từng LE; bỏ qua LE vẫn làm được nhưng Satellite cảnh báo.

## Filter

| Loại | Lọc theo |
|---|---|
| RPM | Tên gói, phiên bản, kiến trúc (gói không modular; không ảnh hưởng SRPM) |
| Package Group | Nhóm gói |
| Erratum (by ID) | Từng advisory |
| Erratum (by Date and Type) | Khoảng ngày issued/updated + loại Security/Bugfix/Enhancement |
| Module Streams | Stream cụ thể (chỉ gói modular) |
| Container Image Tag | Tag image |

- **Include** bắt đầu từ rỗng rồi thêm; **Exclude** bắt đầu từ toàn bộ rồi bỏ. Kết hợp thì Include chạy trước, Exclude chạy sau.
- Filter chỉ có tác dụng với version publish **sau** khi tạo filter.
- Filter làm publish chậm rõ rệt: tài liệu nêu từ vài phút lên khoảng 30 phút; lab thực tế đo được khoảng 30 giây không filter so với khoảng 24 phút có filter errata (CV RHEL 9 khoảng 56 nghìn gói).
- Filter errata theo ngày, Exclude Bugfix + Enhancement, giữ Security là mẫu phổ biến để "đóng băng" PROD mà vẫn nhận RHSA: xem lệnh ở [errata và incremental update](errata-and-incremental-update.md).

```bash
# Ghim phiên bản một gói cho cả môi trường: Exclude các bản mới hơn
hammer content-view filter create --content-view "<CONTENT_VIEW>" --organization "<ORG>" \
  --name "pin-<PACKAGE>" --type rpm --inclusion false
hammer content-view filter rule create --content-view "<CONTENT_VIEW>" --organization "<ORG>" \
  --content-view-filter "pin-<PACKAGE>" --name "<PACKAGE>" --min-version "<NEWER_VERSION>"
# Publish rồi dùng Compare để xác nhận đúng các bản bị loại trước khi promote
```

**Solve dependencies** (theo từng CV hoặc setting **Content View Dependency Solving Default**): chỉ xét gói trong chính CV, làm publish chậm và **ưu tiên dependency hơn luật filter** (có thể kéo lại gói đã chặn). Thường không cần; không bật cho CV có filter bảo mật. Khi dnf báo thiếu dependency do filter, sửa filter thay vì bật solve dependencies.

## Composite Content View

```bash
hammer content-view create --composite --auto-publish yes \
  --name "<CCV_NAME>" --organization "<ORG>"
# Luôn lấy version mới nhất của CV thành phần
hammer content-view component add --composite-content-view "<CCV_NAME>" \
  --component-content-view-id <CV_ID> --latest --organization "<ORG>"
# Hoặc ghim version cụ thể
hammer content-view component add --composite-content-view "<CCV_NAME>" \
  --component-content-view-id <CV_ID> --component-content-view-version-id <CVV_ID> --organization "<ORG>"
hammer content-view publish --name "<CCV_NAME>" --organization "<ORG>"
```

- Publish CV thành phần trước, rồi mới publish CCV (Auto publish làm việc này tự động khi CV thành phần được publish lại).
- CV chỉ dùng làm thành phần của CCV thì không cần promote vào LE.
- Một repository docker không được xuất hiện hai lần trong CCV. Setting **Restrict Composite Content View promotion** (mặc định No) chặn publish/promote CCV khi version thành phần chưa có ở LE đích.

## Rolling Content View

Dành cho host cần nội dung mới nhất hằng ngày nhưng qua Capsule: gán rolling CV cho LE khác Library, Capsule chỉ sync tập con đó.

```bash
hammer content-view create --rolling --name "<ROLLING_CV>" --organization "<ORG>" \
  --repository-ids <REPO_ID_1>,<REPO_ID_2> --lifecycle-environment-ids <LE_ID_1>,<LE_ID_2>
```

## Quản lý version

- **So sánh** hai version (Versions → tick hai version → Compare, chọn View by Different): biết promote sẽ mang lên errata và gói nào; chụp kết quả đính kèm phiếu thay đổi trước khi promote PROD.
- **Xóa** nhiều version: Web UI có wizard yêu cầu chuyển host/AK khỏi version trước khi xóa; Hammer `hammer content-view remove --id <CV_ID> --content-view-version-ids <CVV_ID_1>,<CVV_ID_2>`.
- **Purge** version thừa (chỉ xóa version không thuộc LE nào): `hammer content-view purge --name "<CONTENT_VIEW>" --organization-id <ORG_ID> --versions-to-keep 3`.
- **Distribute archived content view versions** (setting): phát hành repository của version chưa promote tại `/pulp/content/<ORG>/content_views/<CV>/<VERSION>/` để debug; chỉ áp dụng cho version tạo sau khi bật.

## Content view environment (nhiều CV trên một host)

- Setting **Allow multiple content views** (mặc định bật) cho phép host và AK có nhiều content view environment. Tắt setting thì host/AK cũ vẫn giữ, nhưng đăng ký mới với nhiều environment sẽ lỗi.
- **Thứ tự là ưu tiên**: cùng một repository (cùng label) có ở nhiều environment thì host lấy từ environment đứng đầu. Đặt `Library/...` lên đầu sẽ đè các environment khác.
- Hammer và API không có lệnh thêm/bớt lẻ: phải truyền **toàn bộ danh sách theo thứ tự**.

```bash
hammer content-view-environment list --organization "<ORG>"
hammer host update --name "<HOST_FQDN>" --organization-id <ORG_ID> \
  --content-view-environments "DEV/<CONTENT_VIEW>,DEV/<APP_CV>"
hammer activation-key update --name "<ACTIVATION_KEY>" --organization-id <ORG_ID> \
  --content-view-environments "DEV/<CONTENT_VIEW>,DEV/<APP_CV>"
```

Trên host (cần tài khoản Satellite để liệt kê): `subscription-manager environments --list`, `--set <ENV_1>,<ENV_2>`, `--list-enabled`. Bulk: Hosts → All Hosts → chọn host → Manage content → Content view environments (kéo thả để đổi thứ tự).

Nhiều Activation Key lúc đăng ký: khi bật Allow multiple content views, host nhận hợp các environment theo thứ tự key; khi tắt, host nhận environment của key cuối cùng có environment, và key nhiều environment làm đăng ký lỗi.

| So sánh | Composite CV | Nhiều content view environment |
|---|---|---|
| Thời điểm host thấy nội dung | Sau khi publish CCV | Ngay khi gán |
| Xung đột repository trùng | Gộp lúc publish | Theo thứ tự environment |
| Chọn version | Ghim được version cũ của CV thành phần | Luôn là version đang ở LE |

## Host không thấy gói: kiểm tra theo thứ tự (mục 7.2)

1. Repository nằm trong content view environment của host.
2. CV đã được publish **sau** khi thêm repository, và version đó đã promote tới LE của host.
3. Repository không bị filter loại.
4. Repository Enabled (mặc định hoặc override trên AK/host). Repository custom mặc định Disabled.
5. Không bị giới hạn OS/kiến trúc khác host (ưu tiên cao nhất, không override được).
6. Repository Red Hat theo minor release: release của host khớp hoặc không đặt.
7. Capsule phục vụ host đã đồng bộ xong version mới; sau đó trên host `subscription-manager refresh; dnf clean all`.

## Bài học từ triển khai thực tế (ẩn danh)

- Thiết kế đã chạy ổn: một CV cho mỗi bản RHEL major (BaseOS + AppStream + Satellite Client 6), một CV riêng cho OS của Capsule (thêm repository Capsule và Maintenance); LE Library → DEV → PROD; mỗi Capsule chỉ gán DEV và/hoặc PROD; giữ tối đa 3 version mỗi CV.
- Chu kỳ tháng: sync hằng ngày vào Library; đầu tháng publish version mới (Description ghi ngày sync) và promote DEV; 1–2 tuần kiểm thử; promote cùng version lên PROD sau khi phê duyệt thay đổi; cuối kỳ xuất báo cáo errata.
- Nhiều Capsule sau load balancer phải có cùng LE và cùng version CV đã đồng bộ xong; nếu một Capsule sync lỗi, host nhận nội dung không nhất quán (lỗi gói chập chờn). Kiểm tra Infrastructure → Capsules trước khi báo đội vận hành cập nhật host.
- Activation Key chỉ có tác dụng lúc đăng ký: đổi CV/LE cho host đang chạy thì sửa trên host hoặc bulk action ([Activation Key chỉ áp dụng lúc đăng ký](../hosts/activation-key-registration-only.md)).

## Liên quan

- [Errata và incremental update](errata-and-incremental-update.md)
- [Repository Red Hat, custom repository và đồng bộ](repositories-and-sync.md)
- [Subscription manifest và SCA](subscriptions-manifest.md)
- [Đăng ký host](../hosts/host-registration.md) · [Host group và tham số](../hosts/host-groups-and-parameters.md)
- [Capsule sau load balancer](../installation/capsule-load-balancer.md)
- [Satellite Ansible Collection](../ansible/satellite-ansible-collection.md)

## Nguồn

- Red Hat Satellite 6.19, Managing content, chương 7 Content access control for hosts (PDF tr. 57–59) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing content, chương 8 Managing application lifecycles (PDF tr. 60–68) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing content, chương 9 Managing content views and content view environments (PDF tr. 69–100) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing content, mục 12.8–12.10 Multiple activation keys and content view environments (PDF tr. 134–137) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Overview, concepts, and deployment considerations, mục 12.3 Content view scenarios (PDF tr. 53–54); Administering, phụ lục F.7 (PDF tr. 140–143) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- RH403 (Satellite 6.15), chương 2 và 4: environment path, filter Include/Exclude, Composite CV, promote để rollback.
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: thiết kế CV/LE, chu kỳ publish hằng tháng, đo thời gian publish có/không filter trên lab, đồng bộ nhiều Capsule sau load balancer.
