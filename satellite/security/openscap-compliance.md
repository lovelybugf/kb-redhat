---
title: Quét tuân thủ bảo mật với OpenSCAP trên Satellite
tags: [runbook, satellite, compliance, security, capsule, ansible]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Managing security compliance, chương 1–7 (PDF tr. 7–29); Configuring Capsules with a load balancer, chương 10; kinh nghiệm triển khai thực tế (ẩn danh), 2026"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** OpenSCAP trên Satellite gồm bốn bước: bật plugin OpenSCAP trên Capsule, nạp SCAP content khớp phiên bản RHEL của host, tạo compliance policy, rồi triển khai policy lên host bằng Ansible, Puppet hoặc thủ công. Host tự quét theo lịch và gửi báo cáo ARF về OpenSCAP Capsule qua cổng 9090; quét không sửa host, khắc phục là bước riêng cần thử trước ở môi trường không phải production.

## Khái niệm cốt lõi

| Thành phần | Ý nghĩa | Ở đâu trong Satellite |
|---|---|---|
| SCAP content (data stream `*-ds.xml`) | Gói XML chứa rule (XCCDF), kiểm tra lỗ hổng (OVAL); một data stream có nhiều profile | Hosts → Compliance → SCAP contents |
| XCCDF profile | Bộ rule theo một chuẩn: CIS, PCI-DSS, DISA STIG, OSPP, HIPAA... | Chọn khi tạo policy |
| Tailoring file | Tuỳ chỉnh profile (bỏ chọn rule, đổi giá trị) mà không sửa data stream gốc | Hosts → Compliance → Tailoring Files |
| Compliance policy | Lịch quét định kỳ một profile trên tập host (gán qua host group hoặc từng host) | Hosts → Compliance → Policies |
| SCAP client | `foreman_scap_client` trên host: tải content, chạy `oscap`, upload báo cáo | Gói từ repository Satellite Client 6 |
| ARF report | Kết quả quét (Passed / Failed / Othered) | Hosts → Compliance → Reports |
| OpenSCAP Capsule | Capsule nhận báo cáo của host (cổng 9090) và cấp content cho host | Trường OpenSCAP Capsule của host / host group |

Satellite hỗ trợ SCAP 1.2 và 1.3. Một lần quét không thay đổi gì trên host.

## Điều kiện trước khi làm

- Host đã đăng ký; repository BaseOS + AppStream (RHEL 8/9/10) và **Red Hat Satellite Client 6** của đúng phiên bản RHEL đã sync, có trong Content View/Lifecycle Environment của host và được bật trên host (cần để cài SCAP client).
- Host mở được 9090/TCP tới OpenSCAP Capsule (hoặc FQDN của load balancer) — xem [Cổng mạng](../architecture/network-ports.md).
- Quyền: role sẵn có **Compliance viewer** / **Compliance manager**, hoặc từng permission (`view_scap_contents`, `create_scap_contents`, `create_tailoring_files`, `view_policies`, `create_policies`, `edit_policies`, `destroy_policies`, `view_arf_reports`, `destroy_arf_reports`) — xem [Người dùng, role, Organization](users-roles-orgs.md).
- SCAP content phải khớp phiên bản RHEL của host được quét (xem Bước 2).

## Bước 1 — Bật OpenSCAP

Trên Satellite Server plugin đã bật sẵn. Trên **mỗi Capsule** dùng cho OpenSCAP, chạy bằng root:

```bash
satellite-installer \
  --enable-foreman-proxy-plugin-openscap \
  --foreman-proxy-plugin-openscap-ansible-module true \
  --foreman-proxy-plugin-openscap-puppet-module true
```

Trên Satellite, làm mới danh sách tính năng rồi kiểm tra cột Features có `Openscap` (bảng mặc định cắt cột, dùng `--output csv`):

```bash
hammer capsule refresh-features --name "<CAPSULE_FQDN>"
hammer --output csv capsule list --fields Name,Features
```

Web UI: Infrastructure → Capsules → chọn Capsule → feature OpenSCAP ở trạng thái active. Muốn triển khai bằng Puppet thì phải bật Puppet integration trước.

## Bước 2 — Nạp SCAP content

Có ba nguồn, chọn theo phiên bản RHEL của host:

| Cách | Lệnh / thao tác | Khi nào dùng |
|---|---|---|
| Content mặc định (SSG trên chính Satellite) | `hammer scap-content bulk-upload --type default` | Chỉ quét được host cùng **minor** RHEL với Satellite; dùng để nạp lại sau khi SSG trên Satellite được cập nhật |
| SSG đúng minor từ Customer Portal | Tải RPM `scap-security-guide` bản `*.el9_N`, tách `*-ds.xml` bằng `rpm2cpio` rồi upload | Cách được Red Hat khuyến nghị cho từng minor version |
| Lấy từ host tham chiếu (thực tế dự án) | Cài `scap-security-guide` trên một host mỗi bản major, lấy `/usr/share/xml/scap/ssg/content/ssg-rhel<N>-ds.xml` | Môi trường disconnected, có sẵn host RHEL đã đăng ký |

Trên host tham chiếu (root), xem phiên bản và danh sách profile:

```bash
dnf install -y scap-security-guide openscap-scanner
rpm -q scap-security-guide
oscap info --profiles /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

Upload lên Satellite: Hosts → Compliance → SCAP contents → Upload New SCAP Content (Title, file, tab Locations/Organizations → Submit). Bằng CLI trên Satellite:

```bash
# nạp cả thư mục (theo tài liệu hãng)
hammer scap-content bulk-upload --type directory \
  --directory /usr/share/xml/scap/my_content/ \
  --location "<LOCATION>" --organization "<ORG>"

# nạp một file (đã dùng khi upload qua Web UI bị timeout vì file lớn)
hammer scap-content create \
  --title "scap-security-guide-<SSG_VERSION>.el9" \
  --scap-file /root/ssg-rhel9-ds.xml \
  --organizations "<ORG>" --locations "<LOCATION>"

hammer scap-content list --organization "<ORG>" --location "<LOCATION>"
```

Profile hay chọn (phần đuôi của `xccdf_org.ssgproject.content_profile_<ID>`):

| ID | Chuẩn | Ghi chú thực tế |
|---|---|---|
| `cis_server_l1` | CIS Level 1 – Server | Hợp để thí điểm, ít ảnh hưởng vận hành |
| `cis` | CIS Level 2 – Server | ID **không** có đuôi `_l2`; chặt hơn, dễ ảnh hưởng ứng dụng |
| `pci-dss` | PCI-DSS | Cho hệ thống xử lý dữ liệu thẻ |
| `stig` | DISA STIG | Rất chặt |

## Bước 3 — Tailoring file (tuỳ chọn)

Tài liệu hãng dùng SCAP Workbench; trên RHEL có công cụ dòng lệnh `autotailor` (gói `openscap-utils`) tạo ra tailoring file cùng định dạng. Mỗi bản RHEL major nên có tailoring file riêng; một file có thể chứa tuỳ chỉnh của nhiều profile. Trên host tham chiếu (root):

```bash
# xuất danh sách rule ra HTML để đội an toàn thông tin chọn rule cần bỏ
oscap xccdf generate guide \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml > /root/cis-l1-rhel9-guide.html

dnf install -y openscap-utils
autotailor \
  --new-profile-id xccdf_org.ssgproject.content_profile_cis_server_l1_custom \
  --title "CIS L1 Server - <ORG>" \
  --unselect xccdf_org.ssgproject.content_rule_partition_for_tmp \
  --output /root/tailoring-cis-l1-rhel9.xml \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml \
  xccdf_org.ssgproject.content_profile_cis_server_l1
oscap info /root/tailoring-cis-l1-rhel9.xml
```

Mỗi rule cần bỏ thêm một `--unselect`. Upload: Hosts → Compliance → Tailoring Files → New Tailoring File, hoặc:

```bash
hammer tailoring-file create --name "tf-cis-l1-rhel9" \
  --scap-file /root/tailoring-cis-l1-rhel9.xml \
  --organizations "<ORG>" --locations "<LOCATION>"
hammer tailoring-file list
```

## Bước 4 — Tạo compliance policy

Hosts → Compliance → Policies → New Compliance Policy:

1. Deployment Options: **Ansible**, **Puppet** hoặc **Manual**.
2. Tên, mô tả.
3. SCAP Content + XCCDF Profile. Satellite không kiểm tra profile có rule hay không: chọn `Default XCCDF Profile` (rỗng) sẽ ra báo cáo rỗng.
4. (Tuỳ chọn) Tailoring File + XCCDF Profile in Tailoring File — profile trong tailoring ghi đè profile gốc.
5. Lịch: Weekly (chọn thứ), Monthly (chọn ngày) hoặc Custom (biểu thức cron).
6. Locations, Organizations, (tuỳ chọn) Host Groups → Submit.

Ghi lại **Policy ID** (số trong URL `/compliance/policies/<ID>`) nếu dùng phương thức Manual. Xem trước rule: cột Actions → Show Guide.

Sửa policy: Puppet áp dụng ở lần chạy agent kế tiếp (mặc định 30 phút); Ansible phải chạy lại role (hoặc có job định kỳ); Manual phải cập nhật lại cấu hình trên host (Bước 5C).

## Bước 5 — Gán OpenSCAP Capsule và triển khai policy

Đặt **OpenSCAP Capsule** (và Content Source) ở host group để host kế thừa: Configure → Host Groups → host group → tab Host Group → OpenSCAP Capsule → Submit. Capsule đứng sau load balancer: host gửi báo cáo tới FQDN của load balancer cổng 9090 — xem [Capsule sau load balancer](../installation/capsule-load-balancer.md).

### 5A. Ansible (cần Remote Execution)

1. Import role `theforeman.foreman_scap_client` (Configure → Ansible → Roles → Import) — điều kiện Capsule/REX xem [Tích hợp Ansible](../ansible/ansible-integration.md).
2. Host group: chọn OpenSCAP Capsule, tab **Ansible Roles** thêm `theforeman.foreman_scap_client`, tab Parameters đặt biến của role nếu cần → Submit → cột Actions → **Run all Ansible roles**.
3. Từng host: Edit host → OpenSCAP Capsule + role → Submit; về trang Hosts → menu tuỳ chọn → **Legacy UI** → tích host → Select Action → **Assign Compliance Policy** (tích *Remember hosts selection for the next bulk action*) → chọn policy → Submit → Select Action → **Run all Ansible roles**.

### 5B. Puppet

Cần OpenVox agent trên host, Puppet environment có class `foreman_scap_client`; gán class cho host/host group, agent áp dụng ở lần chạy kế tiếp. Chi tiết: Managing security compliance mục 5.5–5.6 (PDF tr. 22–24).

### 5C. Manual — khi không dùng REX (đã chạy thật trên RHEL 10)

Theo bài KB Red Hat "How to set up OpenSCAP policies using the Manual deployment option". Gán policy cho host (Legacy UI → Assign Compliance Policy, hoặc qua host group trong policy), rồi lấy giá trị cấu hình của host trên Satellite:

```bash
hammer host policies-enc --name "<HOST_FQDN>"
```

Kết quả có `Id` (Policy ID), `Profile Id`, `Content path`, `Content download path`, `Tailoring path`, `Tailoring download path`; chuỗi `<DIGEST>` ở cuối là SHA256 của file trên Satellite, Capsule chỉ cấp file khi đường dẫn đúng digest. Trên host (root), cài client (RHEL 8/9 đổi số phiên bản trong tên repository):

```bash
subscription-manager repos --enable=rhel-10-for-x86_64-baseos-rpms \
  --enable=rhel-10-for-x86_64-appstream-rpms \
  --enable=satellite-client-6-for-rhel-10-x86_64-rpms
dnf install -y foreman_scap_client_bash openscap-scanner
ls -l /etc/rhsm/ca/katello-server-ca.pem /etc/pki/consumer/cert.pem /etc/pki/consumer/key.pem
```

Bài KB ghi gói `rubygem-foreman_scap_client`; trên 6.19 repository Satellite Client 6 cung cấp `foreman_scap_client_bash` (cùng file cấu hình, cùng lệnh). Kiểm tra bằng `dnf search scap_client`. Tạo `/etc/foreman_scap_client/config.yaml` (quyền 600), khoá policy (`1:` ở ví dụ) = Policy ID:

```yaml
:server: '<CAPSULE_FQDN>'
:port: 9090
:ca_file: '/etc/rhsm/ca/katello-server-ca.pem'
:host_certificate: '/etc/pki/consumer/cert.pem'
:host_private_key: '/etc/pki/consumer/key.pem'
:fetch_remote_resources: false
1:
  :profile: 'xccdf_org.ssgproject.content_profile_cis_server_l1_custom'
  :content_path: '/var/lib/openscap/content/<DIGEST>.xml'
  :download_path: '/compliance/policies/1/content/<DIGEST>'
  :tailoring_path: '/var/lib/openscap/tailoring/<DIGEST_TAILORING>.xml'
  :tailoring_download_path: '/compliance/policies/1/tailoring/<DIGEST_TAILORING>'
```

Lịch quét bằng cron, năm trường thời gian lấy theo lịch của policy (ví dụ 01:00 thứ Bảy), rồi chạy thử:

```bash
cat > /etc/cron.d/foreman_scap_client_cron << 'EOF'
0 1 * * 6 root /bin/sleep 315; /usr/bin/foreman_scap_client ds 1 2>&1 | logger -t foreman_scap_client
EOF
foreman_scap_client ds 1
```

Kết quả đúng: tải content và tailoring từ `https://<CAPSULE_FQDN>:9090/compliance/policies/1/...`, chạy `oscap xccdf eval`, kết thúc bằng `Report uploaded, report id: <N>`. Khi thay SCAP content hoặc tailoring file, digest đổi theo: chạy lại `policies-enc` và sửa `config.yaml`.

### Tài nguyên từ xa (OVAL) và môi trường disconnected

Một số data stream tham chiếu file OVAL trên Internet; mặc định client bỏ qua nên các rule đó có kết quả `notchecked`. Host ra được Internet: đặt biến Ansible `foreman_scap_client_fetch_remote_resources` = `true` (Puppet: `fetch_remote_resources`). Host không ra Internet (mục 5.2, PDF tr. 19–20): tìm file thiếu bằng `oscap info <ds> | grep WARNING`, tải ở máy có Internet (tên file phải đúng tên data stream tham chiếu), đưa vào custom file repository của Satellite, rồi chép vào `/root` của host (client chạy `oscap` với `--local-files /root`).

## Bước 6 — Quét theo yêu cầu

- Có REX: Hosts → All Hosts → host → **Schedule a job → Run OpenSCAP scan**; kiểm tra thẻ Recent jobs có job "Run scan for all OpenSCAP policies" ở trạng thái succeeded. Quyền: `view_hosts`, `create_job_invocations`, `view_job_invocations`.
- Không có REX: chạy trên host `foreman_scap_client ds <POLICY_ID>`; lịch sử lần quét theo cron: `journalctl -t foreman_scap_client --since "-7 days"`.

## Bước 7 — Xem báo cáo, thống kê, email

- Hosts → Compliance → Reports (hoặc `hammer arf-report list`): số rule Passed / Failed / Othered; bấm vào báo cáo để xem từng rule, Full Report có phần Remediation.
- Ô Search hỗ trợ `and`, `not`, `has`, không hỗ trợ regex. Ví dụ: `failed > 5`, `host ~ prod- and date > "Jan 1, 2026"`, `xccdf_rule_failed = xccdf_org.ssgproject.content_rule_<RULE>` (liệt kê host fail một rule), `xccdf_rule_passed = ...`, `xccdf_rule_othered = ...`.
- Dashboard của policy: Policies → cột Actions → Dashboard; ưu tiên host **Failed** và **Never audited**. Trạng thái Compliance cũng hiện trên trang host.
- Email tóm tắt: tài khoản nhận đăng ký **Compliance policy summary** (Daily/Weekly/Monthly) ở tab Email Preferences, cần SMTP đã cấu hình. Mẫu dùng trong triển khai: tạo tài khoản riêng mang hộp thư nhóm, gán Compliance viewer + role tự tạo có quyền `view_mail_notifications`. Báo cáo dạng file định kỳ (report template "Host - Statuses" có cột Compliance) gửi bằng cron — xem [Report template và cảnh báo CVE](../hosts/report-templates-and-cve-alert.md).

## Bước 8 — Khắc phục (remediation)

Luôn thử ở môi trường không phải production: remediation có thể làm hệ thống ngừng hoạt động.

- **Wizard (cần REX)**: Reports → bấm thời điểm ở cột Reported At → dòng rule failed → Actions → **Remediation**; hoặc chạy job template "Run OpenSCAP remediation - Ansible Default" / "Run OpenSCAP remediation - Script Default". Quyền: `view_arf_reports`, `view_hosts`, `create_job_invocations`. Wizard không có cho mọi rule.
- **Thủ công (không REX)**: Full Report → rule → *Remediation Shell script* → chạy trên host thử → `foreman_scap_client ds <POLICY_ID>` → so sánh báo cáo mới.
- **Rule không áp dụng cho tổ chức**: đội an toàn thông tin duyệt ngoại lệ → bỏ rule trong tailoring file (`autotailor --unselect`) → cập nhật Tailoring File trên Satellite → host quét lại (Manual: lấy lại digest).

## Bước 9 — Xoá báo cáo, xoá policy

- Báo cáo: Hosts → Compliance → Reports → Delete ở từng dòng, hoặc tích nhiều dòng → **Delete reports** (mỗi lần một trang). Xoá toàn bộ: dùng script trong *Using the Satellite REST API* (mục Deleting OpenSCAP reports). Quyền `view_arf_reports`, `destroy_arf_reports`.
- Policy: Policies → cột Actions → Delete (quyền `view_policies`, `destroy_policies`).
- Xoá bản ghi host làm mất báo cáo OpenSCAP và việc gán policy; sau khi đăng ký lại phải gán lại policy và kiểm tra OpenSCAP Capsule — xem [Đăng ký lại host](../hosts/host-reregistration.md).

## Lỗi hay gặp

| Triệu chứng | Nguyên nhân thường gặp | Cách xử lý |
|---|---|---|
| `Policy id <ID> not found` / `Config file could not be loaded` | Sai Policy ID hoặc sai cú pháp YAML | Đối chiếu ID; kiểm tra thụt lề và dấu `:` đầu khoá |
| `Unable to load certs` | Host chưa đăng ký, sai đường dẫn chứng chỉ | `subscription-manager identity`, kiểm tra 3 file chứng chỉ |
| `SCAP content is missing and download failed` | Capsule chưa bật OpenSCAP, sai `download_path`, chặn 9090 | Bước 1; `timeout 5 bash -c "</dev/tcp/<CAPSULE_FQDN>/9090" && echo OPEN` |
| `Untrusted client <UUID> attempted to access /policies/...` | Đường dẫn thiếu hoặc sai digest | Lấy lại `hammer host policies-enc`, sửa config, xoá file lỗi trong `/var/lib/openscap/` |
| `Unable to parse XML at ...` | File tải về chứa thông báo lỗi thay vì XML | Xoá file trong `/var/lib/openscap/content/` và `/tailoring/`, sửa cấu hình, chạy lại |
| `Scan failed` | Dùng ID profile gốc khi policy có tailoring (hoặc ngược lại) | `oscap info <tailoring.xml>` để lấy đúng ID |
| `Upload failed` | Host chưa gán policy, Capsule không phải OpenSCAP Capsule của host, Capsule đang sao lưu offline | Kiểm tra Bước 4–5; chạy lại sau giờ sao lưu |
| Báo cáo rỗng | Chọn profile rỗng (`Default XCCDF Profile`) | Chọn profile có rule |
| Nhiều rule `notchecked` | Rule cần OVAL từ Internet | Mục tài nguyên từ xa ở Bước 5 |
| Đăng ký, dnf và OpenSCAP qua load balancer đều lỗi | Load balancer làm SSL offloading, chặn chứng chỉ client | Load balancer phải TCP pass-through |

## Bài học từ triển khai thực tế (ẩn danh)

- Thí điểm trên một host mỗi bản RHEL major; lấy SCAP content từ chính host đó để khớp phiên bản, đặt tên content kèm phiên bản SSG và giữ một quy ước tên duy nhất trong mọi tài liệu.
- Profile và danh sách rule ngoại lệ do đội an toàn thông tin quyết định; gửi họ file HTML từ `oscap xccdf generate guide`. Khi chưa có quyết định, thí điểm bằng CIS Level 1 – Server.
- Lần quét đầu trên máy cài mặc định, CIS Level 1 cho khoảng 60% rule đạt: dự kiến nhiều rule fail và lập kế hoạch khắc phục theo đợt, bắt đầu bằng rule ít rủi ro (banner, quyền file), tránh rule SSH, xác thực, phân vùng.
- Phương thức Manual chạy tốt khi không bật REX, nhưng mỗi host phải tự cấu hình `config.yaml` + cron và cập nhật lại mỗi khi policy, content hoặc tailoring đổi: số host lớn thì dùng Ansible role.
- Đặt lịch quét lệch khung giờ sao lưu offline của Capsule (upload thất bại khi dịch vụ Capsule dừng).
- Capsule sau load balancer: kiểm thử upload báo cáo khi lần lượt chỉ còn một Capsule trong pool hoạt động.
- Đã kiểm chứng: bật plugin trên Capsule, nạp content/tailoring, policy, triển khai Manual, quét và xem báo cáo qua load balancer. Chưa kiểm chứng trong dự án: triển khai bằng Ansible/Puppet, wizard remediation, email tóm tắt.

## Liên quan

- [Tích hợp Ansible trong Satellite](../ansible/ansible-integration.md)
- [Remote Execution](../hosts/remote-execution.md)
- [Capsule sau load balancer](../installation/capsule-load-balancer.md)
- [Cổng mạng](../architecture/network-ports.md)
- [Report template và cảnh báo CVE](../hosts/report-templates-and-cve-alert.md)
- [Người dùng, role, Organization và Location](users-roles-orgs.md)
- [Huỷ đăng ký và đăng ký lại host](../hosts/host-reregistration.md)
- [Bí mật cần che khi đưa ảnh chụp vào tài liệu](documentation-secrets.md)

## Nguồn

- Red Hat Satellite 6.19, Managing security compliance, chương 1 Security compliance management (PDF tr. 7–8), chương 2 Enabling OpenSCAP (tr. 9), chương 3 Configuring SCAP contents (tr. 10–13), chương 4 Managing compliance policies (tr. 14–17), chương 5 Deploying compliance policies (tr. 18–24), chương 6 Running a scan on demand (tr. 25), chương 7 Monitoring compliance (tr. 26–29) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Configuring Capsules with a load balancer, chương 10 Propagating SCAP content through the load balancer (PDF tr. 36) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Knowledgebase: How to set up OpenSCAP policies using the Manual deployment option — https://access.redhat.com/solutions/6389101
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: phương thức Manual trên RHEL 10 qua load balancer (policies-enc → config.yaml → cron), bảng lỗi `foreman_scap_client`, tạo tailoring bằng `autotailor`, quy trình khắc phục một rule (bước chạy script khắc phục chưa thử).
