---
title: Runbook vá host bằng errata (REX, Ansible, chu kỳ và vá khẩn)
tags: [runbook, satellite, errata, remote-execution, content-view, ansible]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Managing hosts, chương 23 (PDF tr. 158–164), mục 18.10.5 (PDF tr. 114), 19.6.10 (PDF tr. 142–143), 20.8 (PDF tr. 150); RH403 chương 4; kinh nghiệm dự án và lab (ẩn danh)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Vá host trên Satellite: xác định errata Applicable/Installable, đưa bản vá vào Content View của môi trường, áp dụng qua remote execution (Script hoặc Ansible), khởi động lại nếu cần rồi kiểm tra và báo cáo. Satellite không tự cài gì lên host khi promote; mọi thay đổi trên host là một job REX hoặc một lệnh `dnf`.

## Applicable và Installable

| Trạng thái | Nghĩa | Việc phải làm |
|---|---|---|
| Applicable | Host đang cài gói bị errata ảnh hưởng, nhưng bản vá chưa có trong Content View environment của host | Publish version mới rồi promote, hoặc incremental update (vá khẩn) |
| Installable | Bản vá đã nằm trong CV version đang ở Lifecycle Environment của host | Cài ngay bằng job REX hoặc `dnf` |

Loại advisory: RHSA (bảo mật, có severity Critical/Important/Moderate/Low và CVE), RHBA (sửa lỗi), RHEA (tính năng). Không vá được gói không nằm trong repository của Satellite và CV environment của host. Phần CV (publish, promote, incremental update, filter errata) xem [Errata và incremental update](../content/errata-and-incremental-update.md).

## Điều kiện tiên quyết

- [ ] Repository đã sync errata mới nhất; host đăng ký vào một hoặc nhiều CV environment.
- [ ] Remote execution hoạt động trên host: xem [Remote execution](remote-execution.md). Trong lab 6.19 đã kiểm chứng: Capsule bật Ansible, Script, Templates, Registration; host đăng ký có Setup REX, tin khoá SSH của Satellite và các Capsule; bật setting *Prefer registered through Capsule for remote execution*.
- [ ] Capsule của host đã đồng bộ CV version chứa bản vá (Infrastructure > Capsules > `<CAPSULE_FQDN>` > tab Overview/Content).
- [ ] Có khung bảo trì và phê duyệt thay đổi nếu errata cần reboot (cờ *Reboot suggested*).

## Bước 1 — Xác định errata cần vá

Web UI:

- Host: Hosts > All Hosts > `<HOST_FQDN>` > Overview, thẻ **Errata** (Applicable / Installable, tách theo security, bugfix, enhancement); tab **Content > Errata** lọc **Type** và **Severity**.
- Toàn hệ thống: Content > Content Types > Errata, tìm `type = security and severity = Critical` hoặc `cve = <CVE_ID>`; cột Content Host Counts cho biết bao nhiêu host Applicable/Installable.

Hammer (giá trị boolean truyền tường minh):

```bash
hammer erratum list --organization "<ORG>" --errata-restrict-installable true \
  --search "type = security and severity = Critical"
hammer erratum info --id <ERRATUM_ID>
hammer host errata list --host "<HOST_FQDN>" --search "type = security"
hammer host list --organization "<ORG>" --search "applicable_errata = <ERRATUM_ID>"
hammer host list --organization "<ORG>" --search "installable_errata = <ERRATUM_ID>"
```

Đối chiếu trên host — số liệu phải khớp cột Installable (Applicable có thể lớn hơn):

```bash
dnf updateinfo summary
dnf updateinfo list --security
```

Xuất báo cáo trước vá bằng report template errata dựng sẵn: xem [Report template và cảnh báo CVE](report-templates-and-cve-alert.md).

## Bước 2 — Đưa bản vá thành Installable

- Theo chu kỳ: publish CV version mới (Description ghi ngày sync), promote lần lượt Dev → QA → Prod theo [Content View và lifecycle](../content/content-views-and-lifecycle.md).
- Khẩn: incremental update đưa riêng RHSA vào version đang dùng (mục Vá khẩn bên dưới).
- Chờ Capsule đồng bộ xong version mới, trên host chạy `subscription-manager refresh` và `dnf clean all` rồi kiểm tra lại Bước 1.

## Bước 3a — Áp dụng qua REX, Script (đã kiểm chứng trên lab 6.19)

Một host: Hosts > `<HOST_FQDN>` > Content > Errata > lọc > tick RHSA > **Apply** > **Apply via remote execution**. Satellite dùng template *Install errata by search query - Katello Script Default*. Chọn **Apply via customized remote execution** nếu muốn hẹn giờ, chỉnh concurrency.

Nhiều host:

- Content > Content Types > Errata > chọn erratum > tab Content Hosts > tick host > **Apply to Hosts** > Confirm.
- Hosts > Host Collections > `<HOST_COLLECTION>` > Collection Actions > **Errata Installation** > Install Selected (hoặc *via remote execution – customize first*).

Hammer:

```bash
hammer job-invocation create \
  --feature katello_errata_install \
  --inputs errata=<ERRATUM_ID_1>,<ERRATUM_ID_2> \
  --search-query "host_collection = <HOST_COLLECTION>"
```

Search query khác: `name = <HOST_FQDN>`, hoặc `applicable_errata = <ERRATUM_ID>` để vá mọi host còn thiếu erratum đó. Muốn mỗi host một job riêng (lỗi host này không lẫn host khác), tài liệu hãng gợi ý vòng lặp:

```bash
for HOST in $(hammer --csv --csv-separator "|" host list \
      --search "applicable_errata = <ERRATUM_ID>" --organization "<ORG>" \
      | tail -n+2 | awk -F "|" '{ print $2 }'); do
  echo "== Installing on $HOST =="
  hammer job-invocation create --feature katello_errata_install \
    --search-query "name = $HOST" --inputs errata=<ERRATUM_ID>
done
```

## Bước 3b — Áp dụng qua Ansible (đã kiểm chứng trên lab 6.19)

1. Hosts > `<HOST_FQDN>` > Content > Errata > tick RHSA > **Apply** > **Apply via customized remote execution**.
2. Job category **Katello via Ansible**, template **Katello Ansible Default**.
3. **Skip to review** > **Submit**. Kết quả lab: job 1/1 thành công, output Ansible `failed=0`.

Muốn mọi thao tác errata/gói mặc định chạy bằng Ansible: Administer > Remote Execution Features, với mỗi feature có `by_search` trong tên, đổi template từ *Katello Script Default* sang *Katello Ansible Default*. Áp dụng cho cả Web UI lẫn `hammer job-invocation create --feature`. Ansible chỉ chạy được khi Capsule ở chế độ `ssh` và có plugin Ansible.

## Hẹn giờ, chia đợt, chạy thêm lệnh

- Chọn *customize first* > Schedule: **Future execution** cho khung bảo trì, **Recurring** cho vá định kỳ; dùng **Dynamic query** cho job định kỳ để danh sách host được tính lại mỗi lần.
- **Concurrency level** = 1 để vá lần lượt các node sau load balancer; gom host theo phiên bản RHEL chính khi vá hàng loạt (khuyến nghị mục 20.8).
- Chạy lệnh trước/sau khi cài errata: tạo snippet tên đúng bằng tên template cài errata thêm hậu tố ` custom pre` hoặc ` custom post` (ví dụ `Install Errata - Katello Ansible Default custom post`), tab Type chọn Snippet. Bằng Hammer: `hammer template create --file "<SNIPPET_FILE>" --name "<TEMPLATE_NAME> custom pre" --type snippet --organizations "<ORG>" --locations "<LOCATION>"`. Template đang dùng xem ở Administer > Remote Execution Features.

## Bước 4 — Khởi động lại

- Errata có cờ *Reboot suggested* (kernel, glibc, systemd...) hoặc tab **Traces** của host báo cần khởi động lại: xem [Gói và Tracer](packages-and-tracer.md). Trên host kiểm tra nhanh bằng `dnf needs-restarting -r`.
- Reboot qua REX trong khung bảo trì:

```bash
hammer job-invocation create \
  --job-template "Power Action - Script Default" \
  --inputs action=restart \
  --search-query "host_collection = <HOST_COLLECTION>"
```

## Bước 5 — Kiểm tra

1. Monitor > Jobs: job mới nhất thành công đủ N/N host; mở từng host xem output (Ansible: dòng recap `failed=0`).
2. Trang host: erratum vừa vá biến khỏi Installable/Applicable, gói lên đúng bản vá (tab Content > Packages hoặc `rpm -q <PACKAGE>`).
3. Trên host: `dnf history info last` làm bằng chứng cho phiếu thay đổi.
4. Errata chưa biến mất: host chưa gửi lại danh sách gói → chạy `dnf uploadprofile --force-upload` (hoặc `subscription-manager refresh`) rồi tải lại trang.
5. Hammer: `hammer job-invocation list`, `hammer task list`, `hammer task progress --id <TASK_ID>`.
6. Xuất báo cáo sau vá (Installability = Installable) đính kèm biên bản.

## Chu kỳ vá theo môi trường Dev → QA → Prod

Red Hat không ấn định chu kỳ; mô hình dưới đây là đề xuất đã dùng trong một dự án triển khai thực tế, dựa trên phân loại severity của Red Hat:

| Mức | Thời hạn đề xuất | Đường đi content | Cách cài |
|---|---|---|---|
| RHSA Critical | Dev trong 48 giờ, Prod trong 7 ngày | Incremental update vào version hiện hành, promote Dev → QA → Prod | Job REX errata, reboot nếu kernel |
| RHSA Important | Prod trong 30 ngày | Gộp vào đợt publish tháng; incremental nếu CVE đang bị khai thác | Theo đợt vá tháng |
| RHSA Moderate/Low, RHBA, RHEA | Đợt vá tháng hoặc quý | Publish version mới đầu tháng | Theo đợt |

Lịch tháng mẫu:

| Thời điểm | Việc |
|---|---|
| Đầu tháng | Sync, publish CV version mới, promote Dev |
| Tuần 1–2 | Vá Dev, kiểm thử ứng dụng; promote QA, vá QA |
| Tuần 3 | Promote Prod; vá Prod theo nhóm host collection (web, db, batch) trong khung giờ riêng |
| Tuần 4 | Báo cáo errata Installable còn tồn cho bộ phận an ninh |

Chỉ số nên theo dõi: số host còn RHSA Critical Installable (mục tiêu 0 sau 7 ngày), tuổi CV version đang ở Prod (ví dụ không quá 45 ngày), tỷ lệ host báo về Satellite trong 24 giờ.

## Vá khẩn

1. **Incremental update** (cách tài liệu hãng hướng dẫn): thêm RHSA vào CV version đang ở môi trường của host, tạo version phụ x.1, Capsule tự đồng bộ, vá bằng job REX như Bước 3; sau đó đưa cùng nội dung lên các môi trường sau. Đã kiểm chứng trong UAT của một dự án. Giới hạn: version càng cũ càng dễ kéo dependency gây xung đột — khi `dnf` báo lỗi phụ thuộc thì publish version đầy đủ.
2. **Chuyển tạm host sang Lifecycle Environment đã có bản vá**: ghi lại CV/LE hiện tại, gán host sang LE có bản vá, chỉ cài đúng advisory (`dnf update-minimal --advisory <ERRATUM_ID> -y`, không chạy `dnf update`), rồi gán lại ngay. Nhanh nhưng dễ sai (quên gán lại, lỡ cập nhật toàn bộ) và chỉ làm được khi Capsule của host phục vụ LE đó. RH403 mô tả biến thể chuyển host về Default Organization View/Library và khuyên mở case hỗ trợ trước khi dùng.
3. Chỉ cài gói của một CVE/advisory trên host: `dnf update-minimal --cve <CVE_ID>` hoặc `--advisory <ERRATUM_ID>` (chạy qua job *Run Command*).

## Rollback

- Gói: trên host `dnf history list` rồi `dnf history undo <TRANSACTION_ID>` (có thể chạy qua job *Run Command*).
- Nguồn content: promote lại CV version trước vào LE hoặc gán host về version trước — chỉ đổi những gì host *thấy*, không tự hạ gói đã cài.

## Lỗi hay gặp

| Triệu chứng | Nguyên nhân | Xử lý |
|---|---|---|
| Satellite báo Installable nhưng `dnf` không thấy | Cache dnf cũ; Capsule chưa đồng bộ sau promote; host khoá release cũ | `subscription-manager refresh; dnf clean all`; kiểm tra đồng bộ của Capsule; `subscription-manager release --show` |
| Erratum chỉ Applicable, không Installable | Bản vá mới ở Library | Publish/promote hoặc incremental update |
| Job errata lỗi ngay khi kết nối | REX chưa sẵn sàng (khoá SSH, Capsule) | Xem bảng xử lý sự cố ở [Remote execution](remote-execution.md) |
| Job Ansible không chọn được | Capsule thiếu plugin Ansible hoặc đang ở pull mode | Bật plugin, dùng chế độ `ssh` cho Ansible |
| Đã vá bằng `rpm` nhưng Satellite chưa cập nhật | Profile chỉ gửi ở lần chạy dnf/yum kế tiếp | `dnf uploadprofile --force-upload` |
| Vá bằng `dnf update` trên host mang lên gói ngoài phạm vi | Lệnh cập nhật mọi gói có trong LE | Dùng `--advisory`/`update-minimal`, hoặc job errata của Satellite |

## Liên quan

- [Errata và incremental update](../content/errata-and-incremental-update.md) · [Content View và lifecycle](../content/content-views-and-lifecycle.md)
- [Remote execution](remote-execution.md) · [Gói và Tracer](packages-and-tracer.md)
- [Report template và cảnh báo CVE](report-templates-and-cve-alert.md) · [Thông báo email và webhook](../operations/notifications-and-webhooks.md)
- [Activation Key chỉ áp dụng lúc đăng ký](activation-key-registration-only.md) — đổi CV/LE của host đang chạy.
- [RHEL 10 báo GPG check FAILED](../../rhel/rhel10-gpg-release-key-4.md)

## Nguồn

- Red Hat Satellite 6.19, Managing hosts, chương 23 Patching hosts through errata (PDF tr. 158–164); mục 18.10.5 Installing errata on hosts in a host collection (PDF tr. 114); mục 19.6.10 Using Ansible provider for package and errata actions (PDF tr. 142–143); mục 20.8 Best practices for host patching (PDF tr. 150) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- RH403 (Satellite 6.15), chương 4 mục Manage and Apply Errata to Hosts — bảng truy vấn errata, phương án vá khẩn.
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: lab 6.19 vá RHSA qua REX Script và qua Ansible (job 1/1, gói lên bản vá, errata hết, `failed=0`); UAT dự án kiểm chứng errata Installable khớp `dnf updateinfo` và incremental update tạo version x.1 được Capsule tự đồng bộ; bảng chu kỳ vá và phương án chuyển tạm LE lấy từ tài liệu vận hành, đào tạo của dự án.
