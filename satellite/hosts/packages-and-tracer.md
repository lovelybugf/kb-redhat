---
title: Cài, gỡ, cập nhật gói từ Satellite và dùng Tracer
tags: [howto, satellite, remote-execution, hammer, rhel]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Managing hosts, chương 7 (PDF tr. 59–60), chương 20 (PDF tr. 146–150), mục 18.10 (PDF tr. 112–115), chương 24 (PDF tr. 165)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Mọi thao tác gói trên host từ Satellite (cài, nâng cấp, gỡ, bật/tắt repository) đều chạy thành job remote execution, và chỉ dùng được gói có trong Content View environment của host. Tracer (gói `katello-host-tools-tracer`) báo dịch vụ hoặc cả host cần khởi động lại sau khi cập nhật và cho restart qua REX.

## Điều kiện

- Host đã đăng ký, gắn một hoặc nhiều Content View environment chứa gói cần dùng.
- Remote execution chạy được trên host: [Remote execution](remote-execution.md).
- Repository *Red Hat Satellite Client 6* (đúng phiên bản RHEL của host) đã sync, có trong CV environment và đang bật trên host — chứa `katello-host-tools` (gửi danh sách gói/errata về Satellite) và `katello-host-tools-tracer`.

## Bật, tắt repository trên host

Hosts > All Hosts > `<HOST_FQDN>` > tab Content > **Repository sets** > menu tuỳ chọn của repository > **Override to enabled** / **Override to disabled**. Override ghi trên Satellite nên không bị mất khi host refresh; tương tự có thể đặt sẵn trên Activation Key cho host đăng ký sau (xem [Activation Key chỉ áp dụng lúc đăng ký](activation-key-registration-only.md)).

## Xem gói có thể nâng cấp

Web UI: trang host > Content > **Packages** > Status = **Upgradable** (mặc định hiện mọi gói đã cài).

```bash
hammer host list --fields Id,Name
hammer host package list --status upgradable --host-id <HOST_ID>
```

## Cài, nâng cấp, gỡ gói

| Việc | Web UI (trang host > Content > Packages) | Feature REX cho Hammer/API |
|---|---|---|
| Cài | Menu cạnh nút Upgrade > **Install packages** > chọn gói > Install | `katello_package_install` |
| Nâng cấp | Status = Upgradable > chọn phiên bản ở cột *Upgradable to* > tick gói > **Upgrade** | `katello_package_update` |
| Gỡ | Menu tuỳ chọn của gói > **Remove** | `katello_package_remove` |

Mỗi thao tác tạo một job REX và hiện thông báo trên giao diện; theo dõi ở Monitor > Jobs.

```bash
hammer job-invocation create \
  --feature katello_package_install \
  --search-query "name = <HOST_FQDN>" \
  --inputs "package=<PACKAGE_1> <PACKAGE_2>"
hammer job-invocation create \
  --feature katello_package_update \
  --search-query "id = <HOST_ID>" \
  --inputs "package=<PACKAGE>"
```

Nhiều gói ngăn bằng dấu cách. Qua API: POST tới `/api/job_invocations` một JSON `job_invocation` gồm `feature`, `inputs.package`, `search_query`, tuỳ chọn `concurrency_control.concurrency_level`, `scheduling.start_at`/`start_before` (ISO 8601, thiếu múi giờ thì hiểu là UTC) và `targeting_type: dynamic_query` để tính lại danh sách host lúc chạy (mục 20.7, PDF tr. 148–150).

### Hàng loạt theo host collection

Hosts > Host Collections > `<HOST_COLLECTION>` > Collection Actions > **Package Installation, Removal, and Update**: nhập tên gói hoặc package group rồi Install / Update / Remove; **Update All Packages** cập nhật toàn bộ. Mỗi nút có lựa chọn *via remote execution – customize first* để hẹn giờ, chỉnh concurrency (mục 18.10, PDF tr. 112–115).

## Thực hành tốt khi vá gói (mục 20.8)

- Dùng Satellite (Web UI, job template SSH/Ansible) để cài/nâng cấp/gỡ; áp errata qua Satellite thay vì tự tải RPM.
- Khi người dùng cập nhật bằng `dnf` trên host, `katello-host-tools` vẫn gửi danh sách gói để Satellite tính lại errata. Thay đổi bằng `rpm` trực tiếp chỉ được gửi ở lần chạy `dnf`/`yum` kế tiếp; muốn gửi ngay thì chạy `dnf uploadprofile --force-upload`.
- Gom host theo phiên bản RHEL chính khi chạy hàng loạt, nhất là khi nâng cấp gói; chọn *customize first* để đặt thời điểm áp dụng.
- Cần thêm bước riêng (dừng ứng dụng, kiểm tra sau cài) thì sửa hoặc thay job template, không chạy tay ngoài Satellite.
- Host **image mode** (bootc): gói cài thêm chỉ là tạm thời, mất sau reboot — chỉ dùng để gỡ lỗi hoặc vá khẩn, muốn lâu dài thì build image mới (chương 24, PDF tr. 165).

## Tracer: biết cái gì cần khởi động lại

Sau khi cập nhật, tiến trình đang chạy vẫn giữ thư viện cũ trong bộ nhớ. Tracer liệt kê ứng dụng/dịch vụ dùng file đã bị thay và báo khi cả host cần reboot (ví dụ sau khi cập nhật kernel); trên Satellite kết quả hiện ở tab **Traces** của host.

Bật Tracer cho một host:

1. Hosts > All Hosts > `<HOST_FQDN>` > tab **Traces** > **Enable Traces**.
2. Chọn provider (cách cài) cho `katello-host-tools-tracer` > **Enable Tracer**. Satellite chạy một job REX để cài gói; khi xong có thông báo job.

Khởi động lại ứng dụng:

1. Tab **Traces**, tick các ứng dụng cần restart.
2. **Restart app** > **Restart via remote execution** (chạy ngay) hoặc **Restart via customized remote execution** (xem, chỉnh, hẹn giờ job trước khi chạy).

Lưu ý:

- Mục Traces yêu cầu host có trạng thái reboot thì dùng job khởi động lại host trong khung bảo trì (template *Power Action - Script Default*, xem [Vá host bằng errata](patching-with-errata.md)).
- Kiểm tra độc lập trên host (gói `dnf-utils`/`yum-utils` của RHEL, không phụ thuộc Satellite):

  ```bash
  dnf needs-restarting -r
  dnf needs-restarting -s
  ```

  `-r` cho biết có cần reboot; `-s` liệt kê service systemd cần restart. Cách restart service an toàn xem [systemd cơ bản](https://github.com/lovelybugf/kb-base-devops/blob/main/linux/systemd-basics.md).
- Hammer có nhóm lệnh `hammer host traces` (liệt kê, xử lý trace) trong hammer-cli-katello; tài liệu 6.19 không mô tả — chạy `hammer host traces --help` trên bản đang dùng trước khi đưa vào script (chưa kiểm chứng).

## Lỗi hay gặp

| Triệu chứng | Nguyên nhân | Xử lý |
|---|---|---|
| Không thấy gói cần cài trong cửa sổ Install packages | Gói không có trong CV environment của host, hoặc repository đang tắt | Thêm repo vào CV, publish/promote; Override to enabled |
| Job gói lỗi ngay | REX chưa sẵn sàng trên host | Xem [Remote execution](remote-execution.md) |
| Tab Traces trống dù vừa cập nhật | Chưa cài `katello-host-tools-tracer` hoặc thiếu repo Satellite Client 6 | Enable Traces; kiểm tra repo Satellite Client 6 bật trên host |
| Danh sách gói trên Satellite lệch với host | Host chưa gửi profile | `dnf uploadprofile --force-upload` |

## Liên quan

- [Remote execution](remote-execution.md) · [Vá host bằng errata](patching-with-errata.md)
- [Đăng ký host](host-registration.md) · [Content View và lifecycle](../content/content-views-and-lifecycle.md)
- [Loại content khác (module stream, file, container...)](../content/other-content-types.md)

## Nguồn

- Red Hat Satellite 6.19, Managing hosts, chương 7 Restarting applications on hosts through Tracer (PDF tr. 59–60); chương 20 Managing packages in Satellite (PDF tr. 146–150); mục 18.10 Managing host content with host collections (PDF tr. 112–115); chương 24 Package mode and image mode hosts (PDF tr. 165) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: `dnf uploadprofile --force-upload` dùng để Satellite cập nhật ngay danh sách gói sau khi thao tác trên host; `dnf needs-restarting -r` dùng làm bước kiểm tra sau vá.
