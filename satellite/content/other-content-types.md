---
title: "Nội dung ngoài RPM: container, Flatpak, ISO, Ansible collection, file"
tags: [reference, satellite, container-image, registry, ansible, pulp]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Managing content, mục 1.2, 3.8.3–3.8.4, chương 14–18 (PDF tr. 12, 30–31, 147–171); RH403 chương 4; ghi chép thực hành RH403"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Ngoài RPM, Satellite quản lý container image (Satellite làm registry, image đi qua CV/LE), Flatpak (kiểu docker), ISO và file tùy ý (repository loại file), Ansible collection (Satellite làm Galaxy server). Mỗi loại vẫn theo mô hình product → repository → sync/upload → Content View → Lifecycle Environment → Capsule.

## Tổng quan

| Loại | Kiểu repository | Nguồn | Client dùng | Chương (PDF tr.) |
|---|---|---|---|---|
| RPM, module stream, errata | `yum` | Red Hat CDN, repository yum, upload | dnf | 3–4, 13 |
| Kickstart tree | `yum` (Red Hat) | Red Hat CDN | Anaconda khi provisioning | 3, phụ lục A |
| Container image | `docker` | registry.redhat.io, registry khác, push trực tiếp | podman, docker | 14 (147–154) |
| Flatpak | `docker` | Flatpak remote (ví dụ index Flatpak của Red Hat) | flatpak | 15 (155–159) |
| ISO, file tùy ý | `file` | Red Hat CDN (ISO), thư mục có PULP_MANIFEST, upload | curl, wget, provisioning | 16, 18 (160–171) |
| Ansible collection | `ansible collection` | Automation Hub, Galaxy, Satellite khác | ansible-galaxy | 17 (163–165) |

Thực hành tốt: mỗi product chỉ một loại nội dung.

## Container image

- Import: **Content → Products → Repo Discovery**, Repository Type = Container Images, nhập registry, user/password, từ khóa → chọn repository → Create Selected; hoặc tạo repository loại `docker` thủ công. Có thể giới hạn tag cần sync (**Limit Sync Tags**) và xóa manifest không cần.

```bash
hammer product create --name "<PRODUCT>" --organization "<ORG>"
hammer repository create --organization "<ORG>" --product "<PRODUCT>" \
  --name "<REPO_NAME>" --content-type docker \
  --url "https://registry.redhat.io/" --docker-upstream-name "<NAMESPACE>/<IMAGE>" \
  --upstream-username "<USER>" --upstream-password "<PASSWORD>"   # registry.redhat.io cần registry service account
hammer repository synchronize --organization "<ORG>" --product "<PRODUCT>" --name "<REPO_NAME>"
```

- Tên image đổi theo CV/LE (dạng `<org>-<le>-<cv>-<product>-<repo>`); đặt **Registry Name Pattern** ở LE (Content → Lifecycle → Lifecycle Environments, ví dụ `<%= repository.docker_upstream_name %>`) để tên gọn hơn, nhưng pattern phải sinh tên duy nhất toàn cục, nếu không publish/promote sẽ xung đột.
- **Unauthenticated Pull** theo LE: mặc định phải đăng nhập; có thể mở cho PROD, giữ xác thực cho DEV/QA.
- Capsule có nội dung chạy plugin **Container Gateway** làm registry, proxy tới Pulp.
- Client tin CA của Satellite/Capsule (chạy bằng root):

```bash
mkdir -p /etc/containers/certs.d/<SATELLITE_FQDN>
cp <ROOT_CA_FILE> /etc/containers/certs.d/<SATELLITE_FQDN>/ca.crt
# Docker: /etc/docker/certs.d/<SATELLITE_FQDN>/ca.crt
```

- Push/pull (podman login tự sinh personal access token, cần quyền `create_personal_access_tokens`; push cần `edit_products`):

```bash
podman login <SATELLITE_FQDN>
podman push <IMAGE_ID> <SATELLITE_FQDN>/<ORG_LABEL>/<PRODUCT_LABEL>/<REPO_NAME>[:<TAG>]
podman push <IMAGE_ID> <SATELLITE_FQDN>/id/<ORG_ID>/<PRODUCT_ID>/<REPO_NAME>[:<TAG>]
podman search <SATELLITE_FQDN>/
podman pull <SATELLITE_FQDN>/<IMAGE_PATH>[:<TAG>]
```

Giới hạn khi push: chỉ push lên Satellite Server (Capsule nhận qua sync), tên repository chữ thường, chỉ push vào repository do luồng push tạo, pull phải dùng cùng kiểu đường dẫn (label hoặc ID) như lúc push. Một repository docker không được có mặt hai lần trong Composite CV.

## Flatpak

Flatpak repository có kiểu `docker`, đưa vào CV như container và export/import được qua ISS. Ứng dụng cần runtime tương ứng (ví dụ ứng dụng RHEL 10 cần repository `rhel10/flatpak-runtime`).

```bash
hammer flatpak-remote create --name "<REMOTE_NAME>" --organization-id <ORG_ID> \
  --url https://flatpaks.redhat.io/rhel/
hammer flatpak-remote scan --id <REMOTE_ID>
hammer flatpak-remote info --id <REMOTE_ID>
hammer flatpak-remote remote-repository mirror --flatpak-remote-id <REMOTE_ID> \
  --id <FLATPAK_REPO_ID> --product-id <PRODUCT_ID>
```

Sync từ registry.redhat.io cần registry service account. Quyền: `create_flatpak_remotes`, `view_flatpak_remotes`, `edit_flatpak_remotes`. Cài ứng dụng lên host: Managing hosts, phần Flatpak.

## ISO

- ISO của Red Hat: bật repository set dạng `(ISOs)` giống bật repository yum rồi sync.

```bash
hammer repository-set list --product "Red Hat Enterprise Linux Server" --organization "<ORG>" | grep -i iso
hammer repository-set enable --product "<PRODUCT>" --name "<ISO_REPOSITORY_SET>" \
  --releasever <RELEASE> --basearch x86_64 --organization "<ORG>"
```

- ISO tự có: tạo repository loại `file` rồi upload; file lớn hơn 15 MB phải upload bằng Hammer.

```bash
hammer repository create --name "<REPO_NAME>" --content-type file --product "<PRODUCT>" --organization "<ORG>"
hammer repository upload-content --id <REPO_ID> --path ~/image.iso --organization "<ORG>"
```

## Ansible collection

Tạo repository loại **ansible collection**, Upstream URL là một Galaxy endpoint (ví dụ `https://console.redhat.com/api/automation-hub/`, Private Automation Hub, hoặc Satellite khác), tùy chọn nội dung `requirements.yml` để chỉ sync collection cần; mặc định sync cả dependency. Xác thực: Auth Token (và Auth URL SSO với console.redhat.com); sync từ Satellite khác thì để trống.

```yaml
collections:
  - name: <NAMESPACE>.<COLLECTION>
    version: 1.2.3
```

Host dùng Satellite/Capsule làm Galaxy server (cần `ansible-core` từ AppStream):

```ini
[galaxy]
server_list = sat_library, sat_prod

[galaxy_server.sat_library]
url = https://<SATELLITE_FQDN>/pulp_ansible/galaxy/<ORG_LABEL>/Library/custom/<PRODUCT_LABEL>/<REPO_LABEL>/api/

[galaxy_server.sat_prod]
url = https://<CAPSULE_FQDN>/pulp_ansible/galaxy/<ORG_LABEL>/<LE_LABEL>/<CV_LABEL>/custom/<PRODUCT_LABEL>/<REPO_LABEL>/api/
```

```bash
ansible-galaxy collection install <NAMESPACE>.<COLLECTION> --server sat_library
```

## File tùy ý

Repository loại `file` chỉ tạo được trong product custom; dùng để phân phối script, SSH key, SCAP content, ISO nhỏ... và vẫn có version qua CV, nhân bản xuống Capsule.

- Nguồn cục bộ trên Satellite (nhiều file), chạy bằng root:

```bash
satellite-maintain packages install pulp-manifest     # lệnh này dừng dịch vụ và chạy lại installer
mkdir -p /var/lib/pulp/local_repos/<DIR>
satellite-installer --foreman-proxy-content-pulpcore-additional-import-paths /var/lib/pulp/local_repos
pulp-manifest /var/lib/pulp/local_repos/<DIR>        # tạo file PULP_MANIFEST
```

Upstream URL khi đó là `file:///var/lib/pulp/local_repos/<DIR>`. Tránh downtime khi cài gói: `satellite-maintain packages unlock`, cài, rồi `lock`.

- Nguồn từ xa: trên một web server RHEL 9 cài `pulp-manifest`, chạy `pulp-manifest` trong thư mục public, Upstream URL là `http://<WEB_SERVER>/<DIR>`. Thêm/bớt file thì chạy lại `pulp-manifest` và sync lại.
- Không có nguồn: upload trực tiếp; `--path` nhận file, thư mục hoặc glob (glob phải đặt trong nháy).

```bash
hammer repository create --content-type file --name "<REPO_NAME>" \
  --organization-id <ORG_ID> --product "<PRODUCT>" [--url <UPSTREAM_URL>]
hammer repository upload-content --id <REPO_ID> --organization-id <ORG_ID> --path "<PATH_OR_GLOB>"
```

Host tải file qua URL publish, ví dụ trong một CV đã promote: `https://<CAPSULE_FQDN>/pulp/content/<ORG_LABEL>/<LE_LABEL>/<CV_LABEL>/custom/<PRODUCT_LABEL>/<REPO_LABEL>/<FILE>` (repository để Unprotected thì tải bằng HTTP không cần chứng chỉ). Ghi chép thực hành RH403 đã dùng cách này để phân phối gói `.deb` và script cho máy Ubuntu qua CV DEV → TEST → PROD.

## Liên quan

- [Repository Red Hat, custom repository và đồng bộ](repositories-and-sync.md)
- [Content View và Lifecycle Environment](content-views-and-lifecycle.md)
- [Đồng bộ giữa các Satellite](inter-satellite-sync.md)
- [Tích hợp Ansible](../ansible/ansible-integration.md)
- [Boot disk và kickstart offline](../provisioning/boot-disks-and-offline-kickstart.md)

## Nguồn

- Red Hat Satellite 6.19, Managing content, mục 1.2 Content types, 3.8.3–3.8.4 Uploading to file repository (PDF tr. 12, 30–31) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing content, chương 14 Managing container images (PDF tr. 147–154) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing content, chương 15 Managing Flatpak repositories (PDF tr. 155–159) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing content, chương 16 Managing ISO images (PDF tr. 160–162) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing content, chương 17 Managing Ansible content (PDF tr. 163–165) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing content, chương 18 Managing custom file type content (PDF tr. 166–171) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- RH403 (Satellite 6.15), chương 4 (container trong CV, xác thực theo LE); ghi chép thực hành RH403 về repository loại file phân phối gói cho Ubuntu.
