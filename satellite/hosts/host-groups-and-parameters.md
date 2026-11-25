---
title: Host group, tham số host, host collection và system purpose
tags: [concept, satellite, registration, provisioning, hammer]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Managing hosts, chương 3, 5, 18, Phụ lục D (PDF tr. 22–25, 29–34, 109–115, 188); Provisioning hosts, mục 6.3.2; Managing content (multiple activation keys); RH403 chương 3"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Host group là khuôn thiết lập dùng chung (OS, LE/CV, Capsule, Ansible role, tham số), lồng nhau được và nhóm con ghi đè nhóm cha; host collection chỉ là tập host để thao tác hàng loạt. Tham số host kế thừa qua 8 cấp từ Global tới Host, cấp càng gần host càng được ưu tiên; system purpose chỉ phục vụ báo cáo subscription.

## Phân biệt nhanh

| | Host group | Host collection | Activation Key |
|---|---|---|---|
| Mục đích | Khuôn cấu hình cho host | Nhóm host để chạy thao tác hàng loạt | Giá trị mặc định lúc đăng ký |
| Một host thuộc | Đúng 1 host group | Nhiều collection | Có thể đăng ký bằng nhiều AK |
| Lồng nhau | Có (cha/con) | Không | Không |
| Sửa xong, host đã có có đổi? | Không (trừ Puppet class) | Thao tác chạy ngay trên thành viên | Không |
| Dùng ở | Register Host, provisioning, chọn đích REX, Ansible role | Cài/gỡ gói, errata, đổi LE/CV hàng loạt | `subscription-manager register`, Register Host |

AK chi tiết: [Activation Key chỉ áp dụng lúc đăng ký](activation-key-registration-only.md).

## Host group

### Chứa gì và kế thừa thế nào

- Thiết lập thường đặt: Operating system, architecture, partition table, installation media, domain, subnet, Lifecycle Environment, Content View, Content Source (Capsule), OpenSCAP Capsule, Ansible roles, tham số (ví dụ `kt_activation_keys`).
- **Đổi thiết lập host group không lan xuống host đã gán** (chỉ Puppet class được cập nhật). Host nhận giá trị mới khi reprovision; host đang chạy muốn đổi LE/CV thì sửa trên host hoặc thao tác hàng loạt.
- Lúc Register Host, chọn host group sẽ điền sẵn Operating system, Activation Keys và Lifecycle environment; Ansible role của host group chạy ngay sau khi đăng ký.

### Host group lồng nhau

Thứ tự ưu tiên: thiết lập trên host > host group con > host group cha. Trường để trống ở nhóm con thì lấy từ nhóm cha. Ví dụ theo tài liệu hãng (diễn giải):

| Nhóm | Đặt OS | Đặt Ansible role | Host trong nhóm nhận |
|---|---|---|---|
| `Base` (gốc) | RHEL 9.5 | `linux-system-roles.selinux` | — |
| `Base/Webservers` | RHEL 9.6 | (trống) | RHEL 9.6 (từ con) + selinux (từ cha) |
| `Base/Storage` | (trống) | `linux-system-roles.postfix` | RHEL 9.5 (từ cha) + postfix (từ con) |

Thiết kế khuyến nghị: một nhóm gốc chứa thiết lập chung cho cả organization, các nhóm con theo OS, lifecycle environment hoặc vai trò (ví dụ `Base/rhel9/<LIFECYCLE_ENV>`). Tên đầy đủ (title) của nhóm con gồm cả cha, ngăn bằng `/`.

### Tạo host group

Web UI: **Configure → Host Groups → Create Host Group** → chọn **Parent** (nếu kế thừa) → Name → các tab (Ansible Roles, Parameters, Operating System...) → **Submit**.

```bash
hammer hostgroup create \
  --name "<HOST_GROUP>" \
  --organizations "<ORG>" \
  --locations "<LOCATION>" \
  --lifecycle-environment "<LIFECYCLE_ENV>" \
  --content-view "<CONTENT_VIEW>" \
  --content-source-id <CAPSULE_ID> \
  --operatingsystem "<OS>" \
  --architecture "x86_64"
```

Tạo nhóm con: thêm tuỳ chọn chỉ định nhóm cha (`--parent` hoặc `--parent-title`, kiểm `hammer hostgroup create --help`). Tài liệu hãng có script tạo sẵn một host group cho mỗi lifecycle environment bằng vòng lặp trên `hammer lifecycle-environment list` (Managing hosts mục 5.5, PDF tr. 33).

### Gán hoặc đổi host group của host

Web UI: **Hosts → All Hosts** → host → **Edit** → chọn **Host Group** → **Submit**; kiểm tra ở thẻ Details của tab Overview. Hammer (kiểm `--help` trước, chưa chạy thử):

```bash
hammer host update --name "<HOST_FQDN>" --hostgroup-title "<PARENT_HOST_GROUP>/<HOST_GROUP>"
```

## Tham số host (parameters)

### Thứ tự ưu tiên (Phụ lục D)

Host nhận tham số từ các cấp sau, **cấp dưới ghi đè cấp trên**:

| # | Cấp | Đặt ở Web UI |
|---|---|---|
| 1 | Global | Configure → Global Parameters |
| 2 | Organization | Administer → Organizations |
| 3 | Location | Administer → Locations |
| 4 | Domain | Infrastructure → Domains |
| 5 | Subnet | Infrastructure → Subnets |
| 6 | Operating system | Hosts → Provisioning Setup → Operating Systems |
| 7 | Host group (con ghi đè cha) | Configure → Host Groups |
| 8 | Host | Hosts → All Hosts → host → Edit → Parameters |

Ví dụ: `host_registration_insights = false` ở Global nhưng `true` ở một host group thì host đăng ký bằng host group đó vẫn cài Lightspeed.

### Đặt tham số

- Chọn đúng kiểu (string, boolean, integer, real, array, hash, yaml, json): snippet so sánh theo kiểu, ví dụ `host_registration_insights` phải là boolean mới có tác dụng.
- Tuỳ chọn ẩn giá trị chỉ che trên giao diện; template vẫn đọc được giá trị, không dùng tham số để cất mật khẩu.

```bash
hammer global-parameter set --name "<NAME>" --value "<VALUE>" --parameter-type boolean
hammer organization set-parameter --organization "<ORG>" --name "<NAME>" --value "<VALUE>"
hammer location set-parameter --location "<LOCATION>" --name "<NAME>" --value "<VALUE>"
hammer hostgroup set-parameter --hostgroup "<HOST_GROUP>" --name "<NAME>" --value "<VALUE>"
hammer host set-parameter --host "<HOST_FQDN>" --name "<NAME>" --value "<VALUE>"
```

Lệnh `hostgroup set-parameter` có trong tài liệu hãng; các lệnh cấp khác cùng mẫu, kiểm `--help` (nhất là `--parameter-type`) trên bản đang dùng.

### Tham số hay dùng

| Tham số | Cấp hay đặt | Tác dụng |
|---|---|---|
| `host_registration_insights`, `host_registration_insights_inventory` | Global | Bật/tắt Lightspeed và inventory lúc đăng ký — [Tắt Lightspeed](disable-lightspeed.md) |
| `host_registration_remote_execution` | Global/host group | Cài khoá SSH REX lúc đăng ký |
| `host_packages` | Host group | Gói cài thêm lúc đăng ký |
| `remote_execution_ssh_user`, `remote_execution_effective_user_method` | Global/host group | User SSH và cách nâng quyền cho REX |
| `kt_activation_keys` | Host group | AK dùng khi provisioning (kiểu string, nhiều key ngăn bằng dấu phẩy, cùng organization) |
| `fips_enabled` | Host group | `true` để provisioning host theo FIPS |
| `outofsync_interval` | Host | Ghi đè ngưỡng phút coi host "out of sync" (setting chung mặc định 30) |

Đăng ký bằng nhiều AK: key đứng sau thắng khi thiết lập xung đột; thứ tự có thể cố định bằng một tham số host group (Managing content, mục multiple activation keys).

## Host collection

- Tập host tuỳ ý trong một organization (theo chức năng, phòng ban...) để chạy cùng một thao tác: cài/cập nhật/gỡ gói, cài errata, đổi LE/CV. Thao tác gói và errata chạy bằng remote execution, nên host phải chạy được REX.
- Host phải đã đăng ký mới thêm vào collection được; một host thuộc nhiều collection. Có thể đặt giới hạn số host; AK gắn collection sẽ tự thêm host lúc đăng ký, nhưng thất bại nếu collection đã đủ giới hạn.
- Thêm/bớt thành viên **không được ghi vào audit**. Xoá collection không xoá host.
- Đổi LE/CV cho cả collection (Collection Actions → Change assigned Lifecycle Environment or Content View) mất khoảng 4 giờ mới tới host; muốn ngay thì chạy `subscription-manager refresh` trên host (qua REX).

```bash
hammer host-collection create --name "<HOST_COLLECTION>" --organization "<ORG>"
hammer host-collection add-host --name "<HOST_COLLECTION>" --host-ids <ID1>,<ID2> --organization "<ORG>"
hammer host-collection hosts --name "<HOST_COLLECTION>" --organization "<ORG>"
hammer host-collection remove-host --name "<HOST_COLLECTION>" --hosts "<HOST_FQDN>" --organization "<ORG>"
hammer host-collection copy --name "<HOST_COLLECTION>" --new-name "<NEW_NAME>" --organization "<ORG>"
hammer host-collection delete --name "<HOST_COLLECTION>" --organization "<ORG>"
```

Web UI: **Hosts → Host Collections** (tạo, Copy Collection, tab Hosts → Add / List/Remove, tab Collection Actions); trên trang host: thẻ **Host collections → Add host to collections**.

## System purpose

- Thuộc tính: role, usage, service level (SLA), add-ons. Từ Satellite 6.16 chỉ còn Simple Content Access, nên system purpose **không quyết định quyền truy cập nội dung**; nó giúp báo cáo ở dịch vụ Subscriptions trên Hybrid Cloud Console chính xác hơn.
- Đặt ở: Activation Key (áp lúc đăng ký); trang host → tab Overview → thẻ **System purpose → Edit**; hàng loạt: **Hosts → All Hosts** → chọn host → **Manage content → System purpose** (chọn giá trị, *No Change* để giữ, *(unset)* để xoá); trên host:

```bash
subscription-manager syspurpose set usage 'Production'
subscription-manager syspurpose set role 'Red Hat Enterprise Linux Server'
subscription-manager syspurpose
```

- **Release version** trong cùng hộp thoại thì khác: nó khoá host vào một bản minor (chỉ thấy repo của bản đó). Chỉ đặt khi có chủ đích (EUS); bỏ khoá trước khi nâng minor — xem [Nâng cấp RHEL](rhel-major-upgrade-leapp.md).

## Liên quan

- [Đăng ký host bằng global registration](host-registration.md)
- [Activation Key chỉ áp dụng lúc đăng ký](activation-key-registration-only.md)
- [Remote execution](remote-execution.md) · [Vá host bằng errata](patching-with-errata.md)
- [Tổng quan provisioning](../provisioning/provisioning-overview.md)
- [Content View và Lifecycle Environment](../content/content-views-and-lifecycle.md)

## Nguồn

- Red Hat Satellite 6.19, Managing hosts, chương 5 Working with host groups (PDF tr. 29–34) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing hosts, Phụ lục D Host parameter hierarchy (PDF tr. 188) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing hosts, chương 18 Configuring host collections (PDF tr. 109–115); chương 3 system purpose (PDF tr. 22–23); mục 6.2.1 global parameters (PDF tr. 35–36) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Provisioning hosts, mục 6.3.2 (`fips_enabled`) và các bước tham số `kt_activation_keys`; Administering, Phụ lục F.1 (`outofsync_interval`) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Khoá RH403 (Red Hat Satellite 6.15), chương 3 — host collection, giới hạn collection khi đăng ký bằng AK, các lệnh `hammer host-collection`.
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: đặt Content Source/OpenSCAP Capsule sẵn trong host group để người vận hành chỉ cần chọn đúng host group lúc đăng ký; đặt release version hàng loạt qua Manage content → System purpose.
