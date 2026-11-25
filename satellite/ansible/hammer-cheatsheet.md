---
title: Tra cứu nhanh lệnh hammer theo nhóm tác vụ
tags: [reference, satellite, hammer, content-view, activation-key, errata]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Managing content; Managing hosts; Administering Red Hat Satellite; Installing Capsule Server; Installing Satellite Server; Managing security compliance; Configuring authentication (các trang PDF ghi theo từng nhóm); kinh nghiệm triển khai thực tế (ẩn danh), 2026"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Bảng tra lệnh `hammer` hay dùng trên Satellite 6.19, nhóm theo tác vụ từ org/location, content, host, errata, job tới RBAC và OpenSCAP, cú pháp lấy từ tài liệu hãng kèm guide và trang PDF nguồn. Lệnh ghi (TT) là lệnh không có trong guide nhưng đã chạy trong triển khai thực tế; luôn kiểm tra bằng `hammer <lệnh> --help` trên bản đang dùng.

## Mẹo chung

- Thông tin đăng nhập của hammer: `~/.hammer/cli.modules.d/foreman.yml` (`:username:`, `:password:`, `:use_sessions:`); đổi mật khẩu admin thì sửa file này (Configuring authentication 2.4, tr. 11). Không chép file này vào tài liệu.
- Phiên đăng nhập: `hammer auth login negotiate` (Kerberos), `hammer auth login oauth ...` (Keycloak), `hammer auth logout` — xem [Xác thực ngoài](../security/external-authentication.md).
- Định dạng đầu ra (TT): `hammer --output csv <lệnh>` (bảng mặc định cắt cột dài), `--fields Id,Name`, `hammer --no-headers ...` để lấy ID trong script, `--per-page 50`.
- Lọc: `--search '<truy vấn>'` dùng cùng cú pháp ô Search của Web UI (`=`, `~`, `and`, `or`, `not`, `has`).
- Ngữ cảnh: hầu hết lệnh content cần `--organization "<ORG>"` hoặc `--organization-id <ORG_ID>`; lệnh host/infra thêm `--location`.

## Sức khoẻ, Capsule, log

Nguồn: Administering Red Hat Satellite, phụ lục B (PDF tr. 119–122); Installing Capsule Server ch.3 (tr. 23–24).

```bash
hammer ping                                   # trạng thái database và dịch vụ lõi
hammer capsule list
hammer capsule info --id <CAPSULE_ID>
hammer capsule refresh-features --name "<CAPSULE_FQDN>"                 # (TT) sau khi bật plugin
hammer --output csv capsule list --fields Name,Features                 # (TT)
hammer admin logging --list
hammer admin logging --all --level-debug      # trả về: --all --level-production
```

Bổ sung bằng `satellite-maintain service status` và `satellite-maintain health check` — xem [Health check và xử lý sự cố](../operations/health-check-and-troubleshooting.md).

## Organization và Location

Nguồn: Administering Red Hat Satellite ch.6–7 (PDF tr. 37–42).

```bash
hammer organization create --name "<ORG>" --label "<ORG_LABEL>" --description "<DESCRIPTION>"
hammer organization update --name "<ORG>" --compute-resource-ids 1
hammer organization list
hammer organization info --name "<ORG>"                                 # (TT)
hammer organization delete --id <ORG_ID>
hammer location create --name "<LOCATION>" --description "<DESCRIPTION>" --parent-id <PARENT_LOCATION_ID>
hammer location add-organization --name "<LOCATION>" --organization "<ORG>"
hammer location list
hammer location delete --id <LOCATION_ID>
hammer host list --location "<LOCATION>"
```

## Subscription, manifest, nguồn CDN

Nguồn: Installing Satellite Server (connected tr. 23; disconnected tr. 24, 31–35, 49).

```bash
hammer subscription upload --file ~/manifest_file.zip --organization "<ORG>"
hammer subscription refresh-manifest --organization "<ORG>"             # (TT)
hammer subscription list --organization "<ORG>"                         # (TT)
hammer organization configure-cdn --name="<ORG>" --type=redhat_cdn
hammer organization configure-cdn --name="<ORG>" --type=custom_cdn \
  --url https://<CDN_FQDN> --ssl-ca-credential-id "<CA_CREDENTIAL_ID>"
hammer organization configure-cdn --name="<ORG>" --type=export_sync
```

Chi tiết: [Subscription và manifest](../content/subscriptions-manifest.md), [Đồng bộ giữa các Satellite](../content/inter-satellite-sync.md).

## Product, repository, đồng bộ

Nguồn: Managing content ch.2–4 và 6 (PDF tr. 22–56); Installing Capsule Server ch.3 (tr. 24); Managing content ch.8 (tr. 67–68, 74).

```bash
hammer repository-set list --product "Red Hat Enterprise Linux Server" --organization "<ORG>"
hammer repository-set enable --name "Red Hat Enterprise Linux 7 Server (RPMs)" \
  --releasever "7Server" --basearch "x86_64" \
  --product "Red Hat Enterprise Linux Server" --organization "<ORG>"
hammer product create --name "<PRODUCT>" --organization-id <ORG_ID>
hammer product list --organization "<ORG>"
hammer repository create --name "<REPO_NAME>" --content-type "yum" \
  --product "<PRODUCT>" --organization "<ORG>" --url <UPSTREAM_URL> \
  --gpg-key-id <GPG_KEY_ID> --publish-via-http true
hammer repository list --organization "<ORG>" --fields id,name,product
hammer repository update --name "<REPO_NAME>" --product "<PRODUCT>" \
  --organization-label <ORG_LABEL> --download-policy immediate
hammer repository upload-content --id <REPO_ID> --path /path/to/example-package.rpm
hammer content-credentials create --content-type gpg_key --name "<GPG_KEY_NAME>" \
  --organization "<ORG>" --path ~/RPM-GPG-KEY-EXAMPLE
```

Đồng bộ và sync plan:

```bash
hammer product synchronize --name "<PRODUCT>" --organization "<ORG>"
hammer repository synchronize --name "<REPO_NAME>" --product "<PRODUCT>" --organization "<ORG>"
hammer repository synchronize --id <REPO_ID> --skip-metadata-check true   # complete sync
hammer repository synchronize --id <REPO_ID> --validate-contents true     # kiểm tra nội dung yum
hammer sync-plan create --name "<SYNC_PLAN>" --description "<DESCRIPTION>" --enabled true \
  --interval daily --sync-date "2026-01-01 01:00:00" --organization "<ORG>"
hammer sync-plan list --organization "<ORG>"
hammer product set-sync-plan --name "<PRODUCT>" --organization "<ORG>" --sync-plan "<SYNC_PLAN>"
hammer http-proxy create --name <PROXY_NAME> --url http://<PROXY_FQDN>:8080
```

Capsule nhận content:

```bash
hammer capsule content available-lifecycle-environments --id <CAPSULE_ID>
hammer capsule content add-lifecycle-environment --id <CAPSULE_ID> \
  --lifecycle-environment-id <LE_ID> --organization "<ORG>"
hammer capsule content lifecycle-environments --id <CAPSULE_ID>
hammer capsule content synchronize --id <CAPSULE_ID>
hammer capsule content synchronize --id <CAPSULE_ID> --lifecycle-environment-id <LE_ID>
hammer capsule content synchronize --id <CAPSULE_ID> --skip-metadata-check true
```

Chi tiết: [Repository và đồng bộ](../content/repositories-and-sync.md), [Download policy](../content/download-policy.md).

## Lifecycle Environment và Content View

Nguồn: Managing content ch.8–9 (PDF tr. 64–98), ch.13 (tr. 142–143).

```bash
hammer lifecycle-environment create --name "<LIFECYCLE_ENV>" \
  --description "<DESCRIPTION>" --prior "Library" --organization "<ORG>"
hammer lifecycle-environment paths --organization "<ORG>"
hammer lifecycle-environment list --organization "<ORG>"
hammer content-view create --name "<CONTENT_VIEW>" --description "<DESCRIPTION>" \
  --organization "<ORG>" --repository-ids <REPO_ID_1>,<REPO_ID_2>
hammer content-view add-repository --name "<CONTENT_VIEW>" --organization "<ORG>" --repository-id <REPO_ID>
hammer content-view publish --name "<CONTENT_VIEW>" --description "<DESCRIPTION>" --organization "<ORG>"
hammer content-view version list --content-view "<CONTENT_VIEW>" --organization "<ORG>"
hammer content-view version promote --content-view "<CONTENT_VIEW>" --version 1 \
  --to-lifecycle-environment "<LIFECYCLE_ENV>" --organization "<ORG>"
hammer content-view version info --id <CV_VERSION_ID>
hammer content-view info --name "<CONTENT_VIEW>" --organization-id <ORG_ID>
hammer content-view purge --name "<CONTENT_VIEW>" --organization-id <ORG_ID> --versions-to-keep 3
hammer content-view-environment list --organization "<ORG>"
```

Filter errata, Composite CV, incremental update:

```bash
hammer content-view filter create --name "<FILTER>" --type erratum \
  --content-view "<CONTENT_VIEW>" --inclusion false --organization "<ORG>"
hammer content-view filter rule create --content-view "<CONTENT_VIEW>" \
  --content-view-filter "<FILTER>" --start-date "YYYY-MM-DD" \
  --types security,enhancement,bugfix --organization "<ORG>"
hammer content-view create --composite --auto-publish yes --name "<COMPOSITE_CV>" --organization "<ORG>"
hammer content-view component add --component-content-view-id <CV_ID> \
  --composite-content-view "<COMPOSITE_CV>" --latest --organization "<ORG>"
hammer content-view version incremental-update --content-view-version-id <CV_VERSION_ID> \
  --errata-ids <ERRATUM_ID_1>,<ERRATUM_ID_2>
```

Chi tiết: [Content View và Lifecycle](../content/content-views-and-lifecycle.md), [Errata và incremental update](../content/errata-and-incremental-update.md).

## Activation Key

Nguồn: Managing content ch.12 (PDF tr. 132–137).

```bash
hammer activation-key create --name "<ACTIVATION_KEY>" --unlimited-hosts \
  --description "<DESCRIPTION>" --content-view-environments "<LIFECYCLE_ENV>/<CONTENT_VIEW>" \
  --organization "<ORG>"
hammer activation-key update --name "<ACTIVATION_KEY>" --organization "<ORG>" \
  --service-level "Standard" --purpose-usage "Development/Test" \
  --purpose-role "Red Hat Enterprise Linux Server"
hammer activation-key update --id <AK_ID> --organization-id <ORG_ID> \
  --content-view-environments "<LE_1>/<CV_1>,<LE_2>/<CV_2>"
hammer activation-key content-override --name "<ACTIVATION_KEY>" \
  --content-label "<CONTENT_LABEL>" --value 1 --organization "<ORG>"
hammer activation-key product-content --content-access-mode-all true \
  --name "<ACTIVATION_KEY>" --organization "<ORG>"
hammer activation-key add-host-collection --id <AK_ID> \
  --host-collection-id <HOST_COLLECTION_ID> --organization-id <ORG_ID>
hammer activation-key list --organization "<ORG>"                       # (TT)
hammer activation-key info --name "<ACTIVATION_KEY>" --organization "<ORG>"   # (TT)
```

Sửa AK không ảnh hưởng host đã đăng ký — xem [AK chỉ áp dụng lúc đăng ký](../hosts/activation-key-registration-only.md).

## Host, host group, đăng ký

Nguồn: Managing hosts ch.4–6, 18, 20 (PDF tr. 27, 32, 40, 109–112, 147); Managing content ch.9 (tr. 95–98); Configuring Capsules with a load balancer ch.2, 9 (tr. 11, 33).

```bash
hammer host list --organization "<ORG>" --location "<LOCATION>"
hammer host list --fields Id,Name
hammer --output csv host list --fields "Name,Operating System,Content view environment"   # (TT)
hammer host info --name "<HOST_FQDN>"                                    # (TT)
hammer host update --name "<HOST_FQDN>" --organization-id <ORG_ID> \
  --content-view-environments "<LIFECYCLE_ENV>/<CONTENT_VIEW>"
hammer host delete --id <HOST_ID> --location-id <LOCATION_ID> --organization-id <ORG_ID>
hammer host package list --status upgradable --host-id <HOST_ID>
hammer hostgroup create --name "<HOSTGROUP>" --lifecycle-environment "<LIFECYCLE_ENV>" \
  --content-view "<CONTENT_VIEW>" --content-source-id <CAPSULE_ID> \
  --organizations "<ORG>" --locations "<LOCATION>"
hammer hostgroup set-parameter --hostgroup "<HOSTGROUP>" --name <PARAM_NAME> --value "<VALUE>"
hammer hostgroup list --organization "<ORG>"                             # (TT)
hammer host-collection create --name "<HOST_COLLECTION>" --organization "<ORG>"
hammer host-collection add-host --host-ids <HOST_ID_1>,<HOST_ID_2> --id <HOST_COLLECTION_ID>
```

Sinh lệnh đăng ký (chạy lệnh in ra trên host bằng root):

```bash
hammer host-registration generate-command --activation-keys "<ACTIVATION_KEY>"
hammer host-registration generate-command --activation-keys "<ACTIVATION_KEY>" \
  --smart-proxy-id <CAPSULE_ID> --insecure true          # đăng ký qua Capsule/load balancer
# (TT) chọn host group, Capsule, tắt REX/Insights
hammer host-registration generate-command --organization "<ORG>" --location "<LOCATION>" \
  --activation-keys "<ACTIVATION_KEY>" --hostgroup "<HOSTGROUP>" \
  --smart-proxy "<CAPSULE_FQDN>" --setup-remote-execution false --setup-insights false
hammer user registration-token invalidate --user-id <USER_ID>
```

Lệnh in ra chứa token đăng ký: không dán vào tài liệu — xem [Bí mật cần che](../security/documentation-secrets.md). Chi tiết: [Đăng ký host](../hosts/host-registration.md), [Xoá host](../hosts/host-deletion-and-unregister.md).

## Errata

Nguồn: Managing content ch.13 (PDF tr. 140–143); Managing hosts ch.23 (tr. 161).

```bash
hammer erratum list --organization "<ORG>" --errata-restrict-applicable 1
hammer erratum list --organization "<ORG>" --search "type = security"             # (TT)
hammer erratum list --organization "<ORG>" --search "cve = <CVE_ID>"              # (TT)
hammer erratum info --id <ERRATUM_ID>
hammer host list --organization "<ORG>" --search "applicable_errata = <ERRATUM_ID>"   # (TT)
hammer host list --organization "<ORG>" --search "installable_errata = <ERRATUM_ID>"  # (TT)
hammer host errata list --host "<HOST_FQDN>"                                       # (TT)
hammer job-invocation create --feature katello_errata_install \
  --inputs errata=<ERRATUM_ID_1>,<ERRATUM_ID_2> --search-query "name = <HOST_FQDN>"
```

Chi tiết: [Vá host bằng errata](../hosts/patching-with-errata.md).

## Remote Execution (job)

Nguồn: Managing hosts ch.19–20 (PDF tr. 122–147); RH403 ch.7 cho `list`/`output`/`cancel`.

```bash
hammer job-template list
hammer job-template info --id <TEMPLATE_ID>
hammer job-template create --file "<TEMPLATE_FILE>" --job-category "<CATEGORY>" \
  --name "<TEMPLATE_NAME>" --provider-type SSH
hammer job-invocation create --job-template "Run Command - Script Default" \
  --inputs command="uptime" --search-query "hostgroup = <HOSTGROUP>"
hammer job-invocation create --feature katello_package_update \
  --search-query "id = <HOST_ID>" --inputs "package=<PACKAGE>"
hammer job-invocation list
hammer job-invocation output --id <JOB_ID> --host "<HOST_FQDN>"
hammer job-invocation cancel --id <JOB_ID>
```

Chi tiết: [Remote Execution](../hosts/remote-execution.md). Lệnh Ansible role: [Tích hợp Ansible](ansible-integration.md).

## Setting

Nguồn: lệnh `hammer settings set` rải trong Managing content (tr. 51, 54), Managing hosts (tr. 57, 122–124), Installing Capsule Server (tr. 22), Installing Satellite Server (tr. 22, 33).

```bash
hammer settings list --search 'name = unregister_delete_host' --fields Name,Value   # (TT)
hammer settings set --name remote_execution_prefer_registered_through_proxy --value true
hammer settings set --name remote_execution_fallback_proxy --value true
hammer settings set --name remote_execution_global_proxy --value false
hammer settings set --name default_redhat_download_policy --value immediate
hammer settings set --name default_download_policy --value immediate
hammer settings set --name subscription_connection_enabled --value false   # Satellite disconnected
hammer settings set --name=http_proxy --value=<HTTP_PROXY_URL>
hammer settings set --name create_new_host_when_facts_are_uploaded --value false
```

## Task

Nguồn: Administering Red Hat Satellite (tr. 86); Managing hosts (tr. 161); Managing content (tr. 44, 103–106).

```bash
hammer task list --search "state = running"                              # (TT) trước reboot/backup
hammer task list --search "state = paused or result = error" --per-page 20   # (TT)
hammer task list --search "label = Actions::Katello::Repository::Sync and state = running"   # (TT)
hammer task info --id <TASK_ID>
hammer task progress --id <TASK_ID>
```

## Người dùng, role, audit

Nguồn: Administering Red Hat Satellite ch.8–9 (PDF tr. 43–56); Configuring authentication ch.5 (tr. 36).

```bash
hammer user create --login <USER> --mail <EMAIL> --auth-source-id 1 \
  --organization-ids <ORG_ID> --location-ids <LOCATION_ID> --password '<PASSWORD>'
hammer user add-role --id <USER_ID> --role "<ROLE_NAME>"
hammer user-group create --name <USER_GROUP> --role-ids <ROLE_ID_1>,<ROLE_ID_2> --user-ids <USER_ID_1>
hammer user-group add-user --name <USER_GROUP> --user <USER>              # (TT)
hammer role create --name "<ROLE_NAME>"
hammer filter available-permissions
hammer filter create --role "<ROLE_NAME>" --permission-ids <PERM_ID_1>,<PERM_ID_2>
hammer filter create --role "<ROLE_NAME>" --permission-ids <PERM_ID> --search "name ~ ccv*"
hammer role filters --name "<ROLE_NAME>"                                  # (TT)
hammer user ssh-keys add --user-id <USER_ID> --name <KEY_NAME> --key-file ~/.ssh/id_rsa.pub
hammer auth-source external list
hammer audit list --search 'type = host and action = destroy'             # (TT)
```

Chi tiết: [RBAC](../security/users-roles-orgs.md).

## OpenSCAP

Nguồn: Managing security compliance ch.3 (PDF tr. 10–13).

```bash
hammer scap-content list --organization "<ORG>" --location "<LOCATION>"
hammer scap-content bulk-upload --type default
hammer scap-content bulk-upload --type directory --directory /usr/share/xml/scap/my_content/ \
  --organization "<ORG>" --location "<LOCATION>"
hammer scap-content create --title "<TITLE>" --scap-file /root/ssg-rhel9-ds.xml \
  --organizations "<ORG>" --locations "<LOCATION>"                       # (TT)
hammer tailoring-file create --name "<TAILORING_NAME>" --scap-file /root/tailoring.xml \
  --organizations "<ORG>" --locations "<LOCATION>"                       # (TT)
hammer host policies-enc --name "<HOST_FQDN>"                             # (TT) giá trị cho config.yaml
hammer arf-report list                                                    # (TT)
```

Chi tiết: [OpenSCAP](../security/openscap-compliance.md).

## Report template

Nguồn: Managing hosts ch.17 (PDF tr. 97–99).

```bash
hammer report-template list
hammer report-template generate --id <TEMPLATE_ID>
hammer report-template generate --name "Subscription - General Report" --inputs "Days from Now=60"
hammer report-template dump --id <TEMPLATE_ID> > example_export.erb
# (TT) gửi CSV qua email, dùng trong cron trên Satellite
hammer report-template schedule --name "Host - Statuses" --report-format csv --mail-to "<MAILBOX>"
hammer report-template schedule --name "Host - Available Errata" \
  --inputs "Installability=applicable" --report-format csv --mail-to "<MAILBOX>"
```

Chi tiết: [Report template và cảnh báo CVE](../hosts/report-templates-and-cve-alert.md).

## Liên quan

- [Satellite Ansible Collection](satellite-ansible-collection.md) — làm các việc trên bằng playbook
- [Tích hợp Ansible](ansible-integration.md)
- [Người dùng, role, Organization và Location](../security/users-roles-orgs.md)
- [Content View và Lifecycle](../content/content-views-and-lifecycle.md)
- [Health check và xử lý sự cố](../operations/health-check-and-troubleshooting.md)

## Nguồn

- Red Hat Satellite 6.19, Managing content — ch.2–4, 6, 8–9, 12–13 (PDF tr. 22–143) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing hosts — ch.4–6, 17–20, 23 (PDF tr. 27–161) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Administering Red Hat Satellite — ch.6–9 (PDF tr. 37–56), phụ lục B (tr. 119–122) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Installing Capsule Server ch.3 (PDF tr. 22–24); Installing Satellite Server connected/disconnected ch.3, 5 (tr. 22–35, 49); Managing security compliance ch.3 (tr. 10–13); Configuring authentication ch.5 (tr. 35–36) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- RH403 (Satellite 6.15) ch.7: `hammer job-invocation list/output/cancel`, `hammer host errata list`.
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: các lệnh đánh dấu (TT) đã chạy trên Satellite 6.19.
