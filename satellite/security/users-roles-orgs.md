---
title: Organization, Location, người dùng và phân quyền RBAC trong Satellite
tags: [howto, satellite, iam, security, hammer]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Administering Red Hat Satellite, chương 6–9 (PDF tr. 36–57); Managing hosts, chương 6 và 19; kinh nghiệm triển khai thực tế (ẩn danh), 2026"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Organization chia tài nguyên theo đơn vị sở hữu (manifest, content, host), Location nhóm theo vị trí vật lý và có phân cấp; user chỉ thấy và gán được tài nguyên trong Org/Location của mình. Quyền đến từ role, mỗi role là tập filter (permission trên một loại tài nguyên, có thể kèm điều kiện search); nên gán role cho user group thay vì từng user và kiểm thử bằng Impersonate.

## Organization

- Dùng để cô lập tài nguyên và người dùng trong một Satellite: một org cho doanh nghiệp nhỏ; nhiều org cho các đơn vị kinh doanh; org riêng cho từng khách hàng nếu vận hành hộ. Manifest, product, Content View, Activation Key đều thuộc một org.
- **Label** (chữ, số, `_`, `-`, không khoảng trắng) dùng trong đường dẫn nội dung, ví dụ `/pulp/content/<ORG_LABEL>/Library/...`; đặt cẩn thận vì gắn vào URL repository và debug certificate.
- Không xoá **Default Organization** (nơi chứa host chưa gán org; luôn phải còn ít nhất một org). Trước khi xoá org, gỡ lifecycle environment và host group liên quan.

```bash
hammer organization create --name "<ORG>" --label "<ORG_LABEL>" --description "<DESCRIPTION>"
hammer organization update --name "<ORG>" --compute-resource-ids 1
hammer organization list
hammer organization info --name "<ORG>"
hammer organization delete --id <ORG_ID>
```

Web UI: Administer → Organizations → New Organization; sau Submit có thể gán host chưa có org và tài nguyên hạ tầng (subnet, domain, template...). **Debug certificate** của org (Generate and Download) cho phép duyệt `/pulp/content/` bằng trình duyệt hoặc `curl --cert cert.pem --key key.pem` để kiểm tra nội dung — giữ file như một bí mật (mục 6.4–6.5, tr. 37–39).

## Location

- Theo vị trí vật lý/địa lý, có **cấp cha – con**; location con kế thừa thiết lập của cha trừ khi khai báo khác.
- Không xoá **Default Location**.

```bash
hammer location create --name "<LOCATION>" --description "<DESCRIPTION>" --parent-id <PARENT_LOCATION_ID>
hammer location add-organization --name "<LOCATION>" --organization "<ORG>"
hammer location list
hammer location delete --id <LOCATION_ID>
```

Ngữ cảnh làm việc: Web UI chọn ở menu Organization/Location trên thanh trên cùng ("Any Organization" / "Any Location" để xem tất cả); Hammer thêm `--organization`/`--organization-id` và `--location`/`--location-id` vào lệnh. Không thấy user, host hay đối tượng vừa tạo thì kiểm tra ngữ cảnh trước tiên.

## Người dùng

Web UI: Administer → Users → Create User. Tab User chọn **Authorized by** = `INTERNAL` (quản lý trên Satellite) hoặc `EXTERNAL`/nguồn LDAP; tab Organizations/Locations chọn org/location và *Default on login*; tab Roles gán role (hoặc tích Administrator).

```bash
hammer auth-source list
hammer user create --login <USER> --firstname <FIRST_NAME> --lastname <LAST_NAME> \
  --mail <EMAIL> --password '<PASSWORD>' --auth-source-id 1 \
  --organizations "<ORG>" --locations "<LOCATION>" \
  --default-organization "<ORG>" --default-location "<LOCATION>"
hammer user add-role --id <USER_ID> --role "<ROLE_NAME>"
hammer user list
```

`--auth-source-id 1` = xác thực nội bộ; thêm `--admin` để cấp quyền quản trị. Không để mật khẩu thật trong lịch sử shell hay tài liệu.

Các tác vụ khác trên user:

| Việc | Cách làm | Ghi chú |
|---|---|---|
| Impersonate | Administer → Users → cột Actions → Impersonate; thoát bằng biểu tượng impersonation góc trên | Kiểm thử quyền đúng như user thấy; audit ghi hành động dưới tên user bị impersonate |
| Tài khoản chỉ dùng API | User nội bộ có role, đặt mật khẩu ngẫu nhiên (`openssl rand -hex 32`), không dùng mật khẩu đó; tạo Personal Access Token | Dành cho tích hợp, service account |
| Personal Access Token | Tab Personal Access Tokens → Add, đặt Expires; chỉ xem được giá trị một lần | Thử: `curl --user <USER>:<TOKEN> https://<SATELLITE_FQDN>/api/status`; thu hồi bằng Revoke. Quyền `create_personal_access_tokens` |
| SSH key của user | Tab SSH Keys, hoặc `hammer user ssh-keys add --user-id <USER_ID> --name <KEY_NAME> --key-file ~/.ssh/id_rsa.pub` | Satellite cài key này lên host khi provisioning |
| Huỷ registration token | `hammer user registration-token invalidate --user-id <USER_ID>` | Vô hiệu các lệnh đăng ký host đã phát (Managing hosts ch.6) |

## User group

Gom user (và user group khác, tạo phân cấp) để gán role một lần; user group gắn với ngữ cảnh org. Với nguồn xác thực ngoài, user group còn map tới nhóm LDAP/AD/IdM — xem [Xác thực ngoài](external-authentication.md).

```bash
hammer user-group create --name <USER_GROUP> --role-ids <ROLE_ID_1>,<ROLE_ID_2> --user-ids <USER_ID_1>,<USER_ID_2>
hammer user-group add-user --name <USER_GROUP> --user <USER>
hammer user-group list
```

Lấy ID role theo tên để dùng trong script:

```bash
hammer --no-headers role list --search 'name = "Viewer"' --fields Id
```

## Role và filter

- **Role** = tập **filter**; mỗi filter = một loại tài nguyên (Host, Content View, Lifecycle Environment...) + các permission (`view_*`, `create_*`, `edit_*`, `destroy_*`, `promote_or_remove_content_views`...) + (tuỳ chọn) điều kiện search giới hạn trên một số đối tượng. Role không có filter thì không có tác dụng.
- Cách nhanh và an toàn: **Clone** role sẵn có gần nhất (Administer → Roles → Clone), gán org/location cho role, rồi sửa filter. Ví dụ quản trị riêng một org: clone **Organization admin**, chọn org đó.

```bash
hammer role create --name "<ROLE_NAME>"
hammer filter available-permissions
hammer filter create --role "<ROLE_NAME>" --permission-ids <PERM_ID_1>,<PERM_ID_2>
# filter có điều kiện: chỉ content view tên bắt đầu bằng ccv
hammer filter create --role "<ROLE_NAME>" --permission-ids <PERM_ID> --search "name ~ ccv*"
hammer role filters --name "<ROLE_NAME>"
```

Điều kiện search (granular filter):

- Dạng `field toán_tử giá_trị`; logic `and`, `not`, `has`; toán tử `=`, `!=`, `~` (like, không phân biệt hoa thường), `!~`, `^` (in), `!^`, `>`, `>=`, `<`, `<=` (chỉ trường số); ký tự đại diện `_` (một ký tự), `%` (nhiều ký tự).
- Ví dụ: `hostgroup = host-editors` (chỉ host trong host group đó), `name ^ (a, b, c)`, giới hạn theo tên lifecycle environment để cho sửa content ở Dev mà không chạm Production.
- **Điều kiện search không áp dụng cho hành động create**: giới hạn `create_locations` bằng `name = "X"` không ngăn user tạo location tên khác.

Bảng đầy đủ permission → action: chạy `foreman-rake console` và đoạn Ruby ở mục 9.7 (tr. 53–54) để xuất `/tmp/table.html`.

## Role có sẵn hay dùng

| Role | Quyền | Ghi chú |
|---|---|---|
| Administrator (cờ admin) | Toàn quyền | Chỉ cho quản trị và tài khoản khẩn cấp |
| System admin | Settings; quản lý user, user group, role; tạo/gán org, location (không xem tài nguyên bên trong) | Gán được mọi role cho người khác — chỉ cho người tin cậy |
| Organization admin | Mọi quyền trừ quản lý org, gói trong một org | Clone để giao quản trị một org |
| Manager / Site manager | Manager: xem và sửa settings toàn cục; Site manager: bản thu hẹp, quản lý host hạ tầng | Có sẵn `execute_jobs_on_infrastructure_hosts` |
| Viewer | Xem cấu hình, log, báo cáo, thống kê | Chỉ đọc |
| Auditor | Xem audit log | |
| Default role | Xem task, job invocation | Satellite tự gán cho mọi user |
| Register hosts | Sinh lệnh đăng ký host | Managing hosts ch.6 |
| View hosts / Edit hosts | Xem host / thêm, sửa, **xoá** host | |
| Remote Execution User / Manager | Chạy job / toàn bộ tính năng REX | Clone User để giới hạn template và host |
| Compliance viewer / Compliance manager | Xem / quản lý OpenSCAP | Do plugin OpenSCAP cung cấp |
| Content Importer / Content Exporter | Nhập / xuất content (Inter-Satellite Sync) | |
| Ansible Tower Inventory Reader | Đọc inventory cho Ansible Automation Platform | User không phải admin dùng làm dynamic inventory |

Danh sách đầy đủ và filter của từng role: Administer → Roles → Filters.

## Quyền tối thiểu cho tác vụ thường gặp

| Tác vụ | Role / permission tối thiểu | Nguồn |
|---|---|---|
| Đăng ký host bằng lệnh sinh từ Register Host | **Register hosts** (hoặc role tương đương) | Managing hosts ch.6 |
| Đăng ký bằng bootstrap script (cũ) | Edit hosts + Viewer; nếu không chấp nhận quyền xoá host: role tự tạo gồm `view_organizations`, `view_locations`, `view_domains`, `view_hostgroups`, `view_hosts`, `view_architectures`, `view_ptables`, `view_operatingsystems`, `create_hosts` | Managing hosts 6.4.3 (tr. 50) |
| Chạy job REX có giới hạn | Clone **Remote Execution User**, giới hạn `view_job_templates`, `view_hosts`, `view_smart_proxies`, `execute_template_invocation` bằng search; job trên Satellite/Capsule cần thêm `execute_jobs_on_infrastructure_hosts` | Managing hosts 19.2.1 (tr. 118) |
| OpenSCAP | Compliance viewer / manager; quét theo yêu cầu thêm `view_hosts`, `create_job_invocations`, `view_job_invocations`; xoá báo cáo `destroy_arf_reports` | Managing security compliance ch.3–7 |
| Quản lý content (không quản trị hệ thống) | Role tự tạo: filter trên Content View, Product/Repository, Lifecycle Environment, Sync Plan, Activation Key, Host Collection, Organization/Location (view), Capsule content, Task, Host, Subscription | Kinh nghiệm triển khai |
| Nhận email thông báo | Role tự tạo có `view_mail_notifications` + role xem dữ liệu liên quan (Compliance viewer, Viewer) | Kinh nghiệm triển khai |
| Cấu hình nguồn LDAP | `view_authenticators`, `create_authenticators`, `edit_authenticators`, `view/assign_locations`, `view/assign_organizations` | Configuring authentication ch.7 |
| Xem Red Hat Lightspeed | ForemanRhCloud + View hosts | Managing hosts ch.15 |
| Personal Access Token / SSH key | `create_personal_access_tokens`; `create_ssh_key`, `destroy_ssh_key` | Administering 8.2–8.3 |

## Mô hình RBAC mẫu đã kiểm thử (ẩn danh)

Năm user group, mỗi nhóm một vai trò; tài khoản thật chỉ việc cho vào nhóm:

| User group | Role | Được phép | Bị chặn (đã thử) |
|---|---|---|---|
| admins | Administrator | Settings, tạo/xoá user group | — |
| content | Role tự tạo "Content manager" (12 filter như bảng trên) | Publish/promote Content View | Xem/xoá user khác (`Access denied: view_users`) |
| operators | Register hosts + View hosts | Đăng ký host, xem errata | Publish/promote Content View |
| security | Compliance manager | Quản lý OpenSCAP | — |
| viewers | Viewer | Xem | Sửa bất kỳ đối tượng nào |

Kinh nghiệm: kiểm thử từng tài khoản bằng Impersonate hoặc Hammer với tài khoản đó (cùng cơ chế RBAC với Web UI), đối chiếu Monitor → Audits (`hammer audit list`) xem hành động ghi đúng user; xoá hoặc khoá tài khoản kiểm thử sau UAT; `admin` chỉ dùng cho quản trị và khẩn cấp.

## Lưu ý

- User bị giới hạn chặt trong Org/Location được gán: gán role đúng mà vẫn không thấy đối tượng thì thường là thiếu org/location trên user hoặc trên chính đối tượng.
- Role gán qua user group cộng dồn với role gán trực tiếp; gỡ quyền phải kiểm tra cả hai nơi.
- Đổi role của user ngoài (LDAP/AD/IdM) qua user group map ngoài, không sửa thành viên nhóm trên Satellite.

## Liên quan

- [Xác thực ngoài: LDAP, AD, IdM, Keycloak](external-authentication.md)
- [Quét tuân thủ với OpenSCAP](openscap-compliance.md)
- [Remote Execution](../hosts/remote-execution.md)
- [Đăng ký host](../hosts/host-registration.md)
- [Tra cứu nhanh lệnh hammer](../ansible/hammer-cheatsheet.md)

## Nguồn

- Red Hat Satellite 6.19, Administering Red Hat Satellite — chương 6 Managing organizations (PDF tr. 36–39), chương 7 Managing locations (tr. 40–42), chương 8 Managing users and roles (tr. 43–50), chương 9 Creating and managing roles (tr. 51–57) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing hosts — mục 6.4.3 quyền cho bootstrap script (PDF tr. 50), mục 19.2.1 Permissions for remote execution (tr. 118) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: mô hình 5 user group + role "Content manager" tạo bằng hammer, kiểm thử 7/7 thao tác đúng kỳ vọng, audit ghi đúng user.
