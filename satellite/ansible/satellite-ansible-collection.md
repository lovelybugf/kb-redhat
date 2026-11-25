---
title: Tự động hoá cấu hình Satellite bằng collection redhat.satellite
tags: [howto, satellite, ansible, content-view, activation-key]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Using the Satellite Ansible Collection, chương 1–3 (PDF tr. 7–15); RH403 chương 10 Automate Red Hat Satellite Management; tài liệu module theforeman.foreman trên docs.ansible.com"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Collection `redhat.satellite` (gói `ansible-collection-redhat-satellite`, có sẵn trên Satellite) là bộ module gọi REST API để khai báo Organization, repository, Content View, Activation Key, host bằng playbook. Playbook chạy trên `localhost` của bất kỳ máy nào tới được API; thông tin đăng nhập để trong Ansible Vault và truyền một lần qua `module_defaults`.

## Khác gì với tích hợp Ansible trong Satellite

| | Collection `redhat.satellite` | Ansible integration ([trang riêng](ansible-integration.md)) |
|---|---|---|
| Đối tượng tác động | Chính Satellite (cấu hình, content, host record) | Hệ điều hành của host được quản lý |
| Chạy ở đâu | Máy điều khiển bất kỳ, `hosts: localhost`, gọi API qua HTTPS | Capsule SSH vào host (job REX) |
| Dùng cho | Dựng/đồng bộ cấu hình Satellite theo kiểu "infrastructure as code", tác vụ định kỳ (sync, publish, promote) | Giữ cấu hình host bằng role |

Collection upstream tương ứng là `theforeman.foreman` (cộng đồng, không được Red Hat hỗ trợ); module cũ `foreman`/`katello` đã bỏ.

## Cài đặt

- Trên Satellite Server: gói đã cài sẵn.
- Máy khác (chạy bằng root): `dnf install ansible-collection-redhat-satellite` (cần repository cung cấp gói này được bật), hoặc lấy từ Ansible automation hub nếu có subscription AAP: cấu hình `[galaxy_server.automation_hub]` trong `ansible.cfg` với token cá nhân rồi `ansible-galaxy collection install redhat.satellite`.
- Kiểm tra và tra cứu tham số (luôn đối chiếu với bản đã cài, tham số thay đổi giữa các phiên bản):

```bash
ansible-doc --list redhat.satellite
ansible-doc redhat.satellite.content_view_version
```

## Xác thực tới API

Chọn một trong ba cách: username + mật khẩu; username + **Personal Access Token** (đặt token vào trường password); vé Kerberos (`use_gssapi: true`) khi Satellite dùng IdM. Dùng tài khoản riêng chỉ có quyền cần cho playbook — xem [RBAC](../security/users-roles-orgs.md). Không ghi mật khẩu vào playbook hay biến môi trường:

```bash
ansible-vault create sat_vault.yml
```

Nội dung vault (giá trị thật chỉ nằm trong file đã mã hoá):

```yaml
sat_username: <USER>
sat_password: <PASSWORD_OR_TOKEN>
sat_server_url: https://<SATELLITE_FQDN>
sat_http_proxy: http://<PROXY_FQDN>:8080
```

Khung playbook dùng `module_defaults` cho cả nhóm module:

```yaml
- name: Cau hinh Satellite
  hosts: localhost
  gather_facts: false
  vars_files:
    - sat_vault.yml
  environment:
    https_proxy: "{{ sat_http_proxy }}"   # chi khi phai qua HTTP proxy
  module_defaults:
    group/redhat.satellite.satellite:
      username: "{{ sat_username }}"
      password: "{{ sat_password }}"
      server_url: "{{ sat_server_url }}"
  tasks: []
```

Kerberos: bỏ `username`/`password`, thêm `use_gssapi: true`. Chứng chỉ: mặc định `validate_certs: true`; máy điều khiển cần tin CA của Satellite (file `katello-server-ca.crt` ở `https://<SATELLITE_FQDN>/pub/`) hoặc chỉ định bằng `ca_path`, không tắt kiểm tra chứng chỉ ở môi trường thật.

```bash
ansible-playbook --syntax-check --ask-vault-pass site.yml
ansible-playbook --ask-vault-pass site.yml
```

## Module thường dùng (ví dụ task)

Các task dưới đây đặt trong `tasks:` của khung playbook trên.

### Organization, Location, Lifecycle Environment

```yaml
- name: Organization
  redhat.satellite.organization:
    name: "<ORG>"
    label: "<ORG_LABEL>"
    state: present

- name: Location
  redhat.satellite.location:
    name: "<LOCATION>"
    organizations:
      - "<ORG>"

- name: Lifecycle Library -> Dev -> Prod
  redhat.satellite.lifecycle_environment:
    name: "{{ item.name }}"
    prior: "{{ item.prior }}"
    organization: "<ORG>"
  loop:
    - { name: Dev, prior: Library }
    - { name: Prod, prior: Dev }
```

### Repository Red Hat, product và repository tự tạo, sync

```yaml
- name: Bat repository RHEL 9 BaseOS
  redhat.satellite.repository_set:
    name: "Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)"
    product: "Red Hat Enterprise Linux for x86_64"
    organization: "<ORG>"
    repositories:
      - releasever: "9"
    state: enabled

- name: Product tu tao
  redhat.satellite.product:
    name: "<PRODUCT>"
    organization: "<ORG>"

- name: Repository yum tu tao
  redhat.satellite.repository:
    name: "<REPO_NAME>"
    product: "<PRODUCT>"
    organization: "<ORG>"
    content_type: yum
    url: "https://<REPO_URL>/"
    download_policy: on_demand

- name: Sync plan hang ngay
  redhat.satellite.sync_plan:
    name: "daily-sync"
    organization: "<ORG>"
    interval: daily
    enabled: true
    sync_date: "2026-01-01 01:00:00 UTC"
    products:
      - "Red Hat Enterprise Linux for x86_64"
      - "<PRODUCT>"

- name: Sync ngay toan bo product
  redhat.satellite.repository_sync:
    product: "Red Hat Enterprise Linux for x86_64"
    organization: "<ORG>"
```

Repository Red Hat sau khi bật có tên dạng `Red Hat Enterprise Linux 9 for x86_64 - BaseOS RPMs 9` (không có ngoặc, thêm releasever) — dùng tên này khi đưa vào Content View.

### Content View: tạo, publish, promote

```yaml
- name: Content View
  redhat.satellite.content_view:
    name: "<CONTENT_VIEW>"
    organization: "<ORG>"
    repositories:
      - name: "Red Hat Enterprise Linux 9 for x86_64 - BaseOS RPMs 9"
        product: "Red Hat Enterprise Linux for x86_64"
      - name: "<REPO_NAME>"
        product: "<PRODUCT>"

- name: Publish version moi va dua vao Library, Dev
  redhat.satellite.content_view_version:
    content_view: "<CONTENT_VIEW>"
    organization: "<ORG>"
    lifecycle_environments: [Library, Dev]

- name: Promote ban dang o Dev len Prod
  redhat.satellite.content_view_version:
    content_view: "<CONTENT_VIEW>"
    organization: "<ORG>"
    current_lifecycle_environment: Dev
    lifecycle_environments: [Prod]
```

`content_view_version` **không có** `version` và `current_lifecycle_environment` thì mỗi lần chạy publish thêm một version (không idempotent): tách task publish ra playbook chạy theo lịch, playbook cấu hình chỉ dùng `version: "<X.Y>"` hoặc `current_lifecycle_environment`. Module không gỡ version khỏi một environment: promote version khác vào thay.

### Activation Key

```yaml
- name: Activation Key cho RHEL 9 o Dev
  redhat.satellite.activation_key:
    name: "<ACTIVATION_KEY>"
    organization: "<ORG>"
    lifecycle_environment: Dev
    content_view: "<CONTENT_VIEW>"
    content_overrides:
      - label: satellite-client-6-for-rhel-9-x86_64-rpms
        override: enabled
```

- `override`: `enabled`, `disabled` hoặc `default`; `label` là content label của repository.
- Bản collection mới (upstream 5.8.0 trở lên) thêm `content_view_environments` (danh sách) và đánh dấu `lifecycle_environment`/`content_view` là deprecated; `auto_attach` và `subscriptions` không còn ý nghĩa khi dùng SCA. Kiểm tra bằng `ansible-doc redhat.satellite.activation_key` trên bản đã cài.
- AK chỉ áp dụng lúc đăng ký — xem [Activation Key chỉ áp dụng lúc đăng ký](../hosts/activation-key-registration-only.md).

### Host group, host, job

```yaml
- name: Host group
  redhat.satellite.hostgroup:
    name: "<HOSTGROUP>"
    organization: "<ORG>"
    locations: ["<LOCATION>"]
    lifecycle_environment: Dev
    content_view: "<CONTENT_VIEW>"
    content_source: "<CAPSULE_FQDN>"
    openscap_proxy: "<CAPSULE_FQDN>"
    ansible_roles: [baseline]
    parameters:
      - name: host_registration_insights
        value: "false"
        parameter_type: boolean

- name: Chuyen host da dang ky vao host group
  redhat.satellite.host:
    name: "<HOST_FQDN>"
    hostgroup: "<HOSTGROUP>"

- name: Chay lenh tren ca host group qua REX
  redhat.satellite.job_invocation:
    search_query: "hostgroup = <HOSTGROUP>"
    job_template: "Run Command - Script Default"
    inputs:
      command: "uptime"
```

Xoá bản ghi host: `redhat.satellite.host` với `state: absent` (xem [Xoá host và huỷ đăng ký](../hosts/host-deletion-and-unregister.md)). `ansible_roles` cần plugin Ansible; `openscap_proxy` cần plugin OpenSCAP.

### Module khác hay cần

| Module | Dùng để |
|---|---|
| `setting` | Đặt setting, ví dụ `remote_execution_prefer_registered_through_proxy` |
| `registration_command` | Sinh lệnh đăng ký host |
| `content_view_filter`, `content_view_filter_rule` | Filter errata/package của CV |
| `smart_proxy_refresh` | Làm mới feature của Capsule |
| `scap_content`, `scap_tailoring_file` | Nạp SCAP content, tailoring file |
| `user`, `usergroup`, `role`, `auth_source_ldap`, `external_usergroup` | RBAC và xác thực ngoài |
| `subscription_manifest`, `http_proxy`, `global_parameter` | Manifest, proxy, tham số toàn cục |
| `*_info` (`host_info`, `content_view_version_info`...) | Đọc dữ liệu để dùng ở task sau |
| `wait_for_task` | Chờ task Satellite kết thúc |

Collection còn có role dựng sẵn nhận dữ liệu dạng biến (ví dụ `organizations`, `lifecycle_environments`, `repositories`, `content_views`, `activation_keys`, `settings`, `content_view_version_cleanup`) — xem danh sách trong thư mục `roles/` của collection.

## Lưu ý

- Module gọi API với quyền của user: thiếu quyền sẽ báo lỗi 403; nên thử playbook bằng tài khoản thật sẽ dùng trong vận hành.
- Tác vụ dài (sync, publish) chạy đồng bộ cho đến khi task Satellite xong; đặt timeout phù hợp khi chạy từ AAP.
- Giữ playbook trong Git; khi Satellite nâng cấp, đọc lại `ansible-doc` vì tham số có thể bị deprecate.
- Trang này dựa trên tài liệu hãng và RH403 (Satellite 6.15); playbook mẫu chưa chạy trên hệ thống dự án.

## Liên quan

- [Tích hợp Ansible trong Satellite](ansible-integration.md)
- [Tra cứu nhanh lệnh hammer](hammer-cheatsheet.md)
- [Content View và Lifecycle Environment](../content/content-views-and-lifecycle.md)
- [Repository và đồng bộ](../content/repositories-and-sync.md)
- [Người dùng, role, Organization và Location](../security/users-roles-orgs.md)

## Nguồn

- Red Hat Satellite 6.19, Using the Satellite Ansible Collection — chương 1 Introduction, cài đặt và trợ giúp (PDF tr. 7–8), chương 2 Creating a playbook (tr. 9–12), chương 3 Example playbooks (tr. 13–15) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- RH403 (Red Hat Satellite 6.15), chương 10 Automate Red Hat Satellite Management: cài từ automation hub hoặc RPM, FQCN, ví dụ product/repository.
- Tài liệu module upstream `theforeman.foreman` (activation_key, content_view_version, hostgroup, host) — https://docs.ansible.com/ansible/latest/collections/theforeman/foreman/index.html (đối chiếu 2026-11-25).
