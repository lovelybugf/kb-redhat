---
title: Cài Capsule Server 6.19
tags: [runbook, satellite, capsule, installation, certificate]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Installing Capsule Server, chương 1–3 (PDF tr. 7–25); RH403 chương 6; kinh nghiệm lab Satellite/Capsule 6.19.5 trên RHEL 9.8 và triển khai thực tế (ẩn danh)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Cài Capsule theo thứ tự: tin CA của Satellite, đăng ký OS bằng Activation Key, cài `satellite-capsule`, sinh tarball bằng `capsule-certs-generate` trên Satellite, chạy lệnh installer mà nó in ra. Cuối cùng gán Organization, Location, Lifecycle Environment và sync nội dung.

## Điều kiện

- RHEL 9 bản minor mới nhất, cài `@Base`, chưa hardening; ≥ 4 CPU, ≥ 12 GB RAM, swap ≥ 4 GB; FQDN chữ thường; SELinux không disabled; NTP; UTF-8 (chương 1, PDF tr. 7–8).
- Phiên bản Capsule phải trùng Satellite (Capsule 6.19 không đăng ký được vào Satellite 6.18).
- Không đăng ký Capsule vào Red Hat CDN; Capsule lấy gói từ Satellite.
- Satellite và Capsule phân giải DNS thuận/nghịch của nhau; không có HTTP proxy giữa hai bên (không hỗ trợ).
- Dung lượng: `/var/lib/pulp` 300 GB, `/var/lib/pgsql` 20 GB (trên Capsule có thể lớn gấp 2–3 lần pgsql của Satellite khi nhiều LE/CV), `/usr` 3 GB, `/opt/puppetlabs` 500 MB (mục 1.3).
- Cổng tối thiểu: Capsule → Satellite 443 (và 80 để tải CA); Satellite → Capsule 443, 9090; host → Capsule 443, 80, 8000, 9090. Đầy đủ: [Cổng mạng](../architecture/network-ports.md).
- Phần chuẩn bị OS giống Satellite: [Cài Satellite connected](install-satellite-connected.md), Bước 1.

Chuẩn bị trên Satellite (mục 2.1.1):

- Manifest đã import; đã sync `rhel-9-for-x86_64-baseos-rpms`, `rhel-9-for-x86_64-appstream-rpms`, `satellite-capsule-6.19-for-rhel-9-x86_64-rpms`, `satellite-maintenance-6.19-for-rhel-9-x86_64-rpms` (thêm Satellite Client 6 cho RHEL 9).
- Một Content View riêng cho Capsule chứa các repo trên, đã publish và promote; một Activation Key trỏ tới CV/LE đó. Repo Capsule, Maintenance, Client không tự bật theo release: trong tab Repository Sets của AK chọn **Override to Enabled**. Xem [Content View và Lifecycle](../content/content-views-and-lifecycle.md), [Activation Key chỉ áp dụng lúc đăng ký](../hosts/activation-key-registration-only.md).

## Bước 1. Capsule tin CA của Satellite

Không bật Insecure khi đăng ký; thay vào đó cài CA và đối chiếu dấu vân tay. Trên Capsule (root):

```bash
curl -s -o /etc/pki/ca-trust/source/anchors/katello-server-ca.crt \
  http://<SATELLITE_FQDN>/pub/katello-server-ca.crt
openssl x509 -noout -subject -fingerprint -sha256 \
  -in /etc/pki/ca-trust/source/anchors/katello-server-ca.crt
update-ca-trust extract
curl -sI https://<SATELLITE_FQDN> | head -1
```

Trên Satellite lấy dấu vân tay gốc; hai chuỗi SHA256 phải trùng:

```bash
openssl x509 -noout -fingerprint -sha256 -in /var/www/html/pub/katello-server-ca.crt
```

## Bước 2. Đăng ký OS Capsule vào Satellite

Hosts → Register Host: chọn Organization, Location, Activation Key; tab Advanced tắt Setup REX và Setup Insights nếu không dùng; Generate. Hoặc bằng hammer trên Satellite, rồi **truyền lệnh qua stdin** sang Capsule:

```bash
hammer host-registration generate-command \
  --organization "<ORG>" --location "<LOCATION>" \
  --activation-keys "<ACTIVATION_KEY>" \
  --setup-insights false --setup-remote-execution false \
  | ssh root@<CAPSULE_FQDN> 'bash -s'
```

- Kinh nghiệm lab: nhét lệnh đăng ký vào trong dấu nháy của `ssh host "..."` làm hỏng URL/token và nhận 401; truyền qua stdin (`| ssh ... 'bash -s'`) thì chạy đúng.
- Token trong lệnh có hạn (mặc định 4 giờ); hết hạn thì sinh lại. Không chụp/ghi token vào tài liệu.
- Máy từng đăng ký nơi khác: `subscription-manager unregister; subscription-manager clean` trước, hoặc thêm `--force true`. Chi tiết: [Đăng ký lại host](../hosts/host-reregistration.md).

Kiểm tra trên Capsule:

```bash
subscription-manager identity
subscription-manager config --list | grep -E "hostname|baseurl"
```

## Bước 3. Repository và gói

```bash
subscription-manager repos --disable "*"
subscription-manager repos \
  --enable=rhel-9-for-x86_64-baseos-rpms \
  --enable=rhel-9-for-x86_64-appstream-rpms \
  --enable=satellite-capsule-6.19-for-rhel-9-x86_64-rpms \
  --enable=satellite-maintenance-6.19-for-rhel-9-x86_64-rpms
dnf repolist enabled
dnf upgrade -y
dnf needs-restarting -r
dnf install -y satellite-capsule
```

Có kernel mới sau `dnf upgrade` thì reboot trước khi chạy installer. Mở firewalld (tài liệu hãng mở thêm dns/dhcp/tftp/puppetmaster cho provisioning và Puppet; chỉ mở thứ dùng):

```bash
firewall-cmd --add-service=https --add-service=http --add-port=9090/tcp --add-port=8000/tcp
firewall-cmd --runtime-to-permanent
```

## Bước 4. Sinh chứng chỉ trên Satellite

```bash
mkdir -p /root/capsule_cert
chmod 700 /root/capsule_cert
capsule-certs-generate \
  --foreman-proxy-fqdn <CAPSULE_FQDN> \
  --certs-tar /root/capsule_cert/<CAPSULE_FQDN>-certs.tar
```

- Thư mục đích phải tồn tại trước: khi triển khai đã gặp trường hợp lệnh báo `Success!` nhưng không có file tar.
- Lệnh in ra một khối `satellite-installer --scenario capsule ...` kèm OAuth consumer key/secret: đây là bí mật, lệnh là duy nhất cho từng Capsule. Không copy khối lệnh từ màn hình (dấu `\` sát giá trị dễ hỏng khi dán); tách ra file chỉ root đọc được.
- Chứng chỉ của CA doanh nghiệp: thêm `--server-cert`, `--server-key`, `--server-ca-cert` (xem [Chứng chỉ SSL](certificates.md)). Capsule sau load balancer: thêm `--foreman-proxy-cname` (xem [Capsule sau load balancer](capsule-load-balancer.md)) — phải làm ngay lần cài đầu.

## Bước 5. Chạy installer trên Capsule

Chép tarball sang Capsule vào đúng đường dẫn ghi trong `--certs-tar-file` (đặt ở `/root`, không đặt trong `/tmp` hay `/var/tmp`):

```bash
scp /root/capsule_cert/<CAPSULE_FQDN>-certs.tar root@<CAPSULE_FQDN>:/root/<CAPSULE_FQDN>-certs.tar
```

Trên Capsule, chạy lệnh đã lưu (dạng chuẩn):

```bash
satellite-installer --scenario capsule \
  --certs-tar-file "/root/<CAPSULE_FQDN>-certs.tar" \
  --foreman-proxy-foreman-base-url "https://<SATELLITE_FQDN>" \
  --foreman-proxy-trusted-hosts "<SATELLITE_FQDN>" \
  --foreman-proxy-trusted-hosts "<CAPSULE_FQDN>" \
  --foreman-proxy-oauth-consumer-key "<OAUTH_KEY>" \
  --foreman-proxy-oauth-consumer-secret "<OAUTH_SECRET>"
```

Kết quả đúng: `Success!` và `Capsule is running at https://<CAPSULE_FQDN>:9090`; log ở `/var/log/foreman-installer/capsule.log`. Lỗi 401 ở bước đăng ký Capsule với Satellite: [Capsule installer lỗi 401 OAuth](capsule-oauth-401.md).

**Không xoá tarball** sau khi cài: cần cho nâng cấp và cài lại Capsule; đưa thư mục `/root/capsule_cert` trên Satellite vào phạm vi sao lưu.

## Bước 6. Trusted proxies trên Satellite

Bắt buộc khi host đăng ký và provisioning qua Capsule: Satellite chỉ tin header `X-Forwarded-For` từ localhost (mục 3.1). Lệnh ghi đè cả danh sách, nên liệt kê lại localhost và mọi Capsule đã có; chỉ dùng IP cụ thể, không dùng dải quá rộng:

```bash
satellite-installer \
  --foreman-trusted-proxies "127.0.0.1/8" \
  --foreman-trusted-proxies "::1" \
  --foreman-trusted-proxies "<CAPSULE_IP>"
satellite-installer --full-help | grep -A 2 "trusted-proxies"
```

## Bước 7. Gán Organization và Location

```bash
hammer capsule update --name "<CAPSULE_FQDN>" \
  --organizations "<ORG>" --locations "<LOCATION>"
```

Web UI (mục 2.6): chọn Any Organization / Any Location → Hosts → All Hosts → chọn Capsule → Change associations → Organization (tick **Fix on mismatch**), tương tự Location; rồi Administer → Organizations / Locations → tab Capsules phải có Capsule trong Selected items.

## Bước 8. Gán Lifecycle Environment và sync

```bash
hammer capsule content add-lifecycle-environment --name "<CAPSULE_FQDN>" \
  --lifecycle-environment "<LIFECYCLE_ENV>" --organization "<ORG>"
hammer capsule content synchronize --name "<CAPSULE_FQDN>"
hammer capsule content info --name "<CAPSULE_FQDN>" --organization "<ORG>"
```

- Không gán **Library** cho Capsule: mỗi lần Satellite sync CDN sẽ kéo theo sync Capsule tự động, tốn tài nguyên, băng thông và đĩa (mục 3.3).
- Web UI: Infrastructure → Capsules → Overview → Synchronize → Optimized Sync hoặc Complete Sync (Complete Sync = `--skip-metadata-check true`).
- Download policy của Capsule: [Download policy](../content/download-policy.md).

## Kiểm tra

Trên Capsule:

```bash
satellite-maintain service list
satellite-maintain health check
```

Trên Satellite:

```bash
hammer capsule refresh-features --name "<CAPSULE_FQDN>"
hammer capsule info --name "<CAPSULE_FQDN>"
hammer capsule list
```

Feature mong đợi gồm Pulpcore, Registration, Templates, Logs, Container_Gateway, Dynflow, Script; Openscap chỉ có sau khi bật plugin. Đăng ký thử một host qua Capsule: [Đăng ký host](../hosts/host-registration.md).

## Tuỳ chọn sau cài

- REX pull-mqtt trên Capsule: `satellite-installer --foreman-proxy-plugin-remote-execution-script-mode pull-mqtt`, mở `mqtt` (1883), đặt `remote_execution_prefer_registered_through_proxy` = true (mục 3.2). Xem [Remote execution](../hosts/remote-execution.md).
- BMC: `--foreman-proxy-bmc "true" --foreman-proxy-bmc-default-provider "freeipmi"` (mục 3.5).
- Puppet: một Capsule chịu tối đa khoảng 100 agent chạy đồng thời (quá thì HTTP 503), Phụ lục A.

## Lỗi hay gặp

| Hiện tượng | Nguyên nhân | Xử lý |
|---|---|---|
| Installer Capsule báo 401 khi đăng ký với Satellite | OAuth key/secret trong lệnh khác giá trị Foreman đang dùng | [Capsule installer lỗi 401 OAuth](capsule-oauth-401.md) |
| Lệnh đăng ký OS báo 401 | Token hết hạn hoặc bị hỏng do quoting | Sinh lại, truyền qua stdin |
| `curl: (60) SSL certificate problem: self-signed certificate` khi đăng ký | Capsule chưa tin CA | Làm lại Bước 1 |
| Lỗi SSL giữa Capsule và Satellite | Lệch giờ; tarball sinh trên Satellite khác | `chronyc tracking`; sinh lại tarball trên đúng Satellite |
| Capsule sync lỗi | Chặn 443 Capsule → Satellite | Mở 443 trực tiếp, không qua proxy |
| Satellite không điều khiển được Capsule | Chặn 9090 Satellite → Capsule | Mở 9090, Refresh features |
| Host đăng ký qua Capsule hiện sai IP | Chưa khai báo trusted proxies | Bước 6 |
| `dnf` trên Capsule báo "could not load PEM client certificate" | `redhat.repo` bị ghi đè do dán nhầm lệnh dành cho host (khi gửi lệnh hàng loạt) | Sao lưu và xoá `redhat.repo`, `subscription-manager refresh`, bật lại 4 repo |

## Liên quan

- [Capsule installer lỗi 401 OAuth](capsule-oauth-401.md) · [Capsule sau load balancer](capsule-load-balancer.md) · [Chứng chỉ SSL](certificates.md) · [Nâng cấp và cập nhật](upgrade-satellite.md)
- [Thành phần và dịch vụ](../architecture/components-and-services.md) · [Sao lưu và khôi phục](../operations/backup-and-restore.md)

## Nguồn

- Red Hat Satellite 6.19, Installing Capsule Server, chương 1 Preparing your environment (PDF tr. 7–10), chương 2 Installing Capsule Server (PDF tr. 11–20), chương 3 Additional configuration (PDF tr. 21–25), Phụ lục A (PDF tr. 26) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- RH403, chương 6 Deploy Satellite Capsule Servers (quy trình và cách giải thích).
- Kinh nghiệm lab Satellite/Capsule 6.19.5 trên RHEL 9.8 (ẩn danh), 2026: thứ tự tin CA → đăng ký AK qua stdin → `dnf update` → cài `satellite-capsule` → reboot nếu có kernel mới → installer → gán Org/Location/LE → sync đã chạy thành công.
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: đối chiếu dấu vân tay CA, override repo trong AK, thư mục tarball phải có sẵn, `redhat.repo` của Capsule bị ghi đè.
