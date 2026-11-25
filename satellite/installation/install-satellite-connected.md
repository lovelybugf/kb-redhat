---
title: Cài Satellite Server 6.19 có kết nối Internet (connected)
tags: [runbook, satellite, installation, rhel, networking]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Installing Satellite Server in a connected network environment, chương 1–3 và 5 (PDF tr. 8–23, 26–46); RH403 ghi chép quy trình cài đặt; kinh nghiệm triển khai thực tế (ẩn danh)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Cài Satellite connected gồm sáu bước: chuẩn bị RHEL 9 sạch, đăng ký RHSM và bật 4 repo, cài gói `satellite`, chạy `satellite-installer --scenario satellite`, import manifest, rồi kiểm tra bằng `hammer ping`. Hai điều kiện hay làm hỏng cài đặt nhất là FQDN không phân giải đúng hai chiều và đồng hồ lệch giờ.

## Khi nào dùng

- Máy Satellite ra được Red Hat CDN (trực tiếp hoặc qua HTTP proxy): lấy gói cài đặt và nội dung từ CDN.
- Không có Internet → [Cài Satellite disconnected](install-satellite-disconnected.md). Cài Capsule → [Cài Capsule Server](install-capsule.md).
- Quyết định kiến trúc, cỡ máy trước khi cài: [Lập kế hoạch triển khai](../architecture/deployment-planning.md), [Sizing và tuning](../architecture/sizing-and-tuning.md).

## Yêu cầu hệ thống (Installing connected, chương 1, PDF tr. 8–11)

| Hạng mục | Yêu cầu |
|---|---|
| Hệ điều hành | RHEL 9 x86_64, bản minor mới nhất tại thời điểm cài (không hỗ trợ EUS); cài nhóm `@Base`, không phần mềm bên thứ ba, không hardening trước khi cài |
| Tài nguyên | ≥ 4 CPU core, ≥ 20 GB RAM, swap ≥ 4 GB |
| Hostname | FQDN chữ thường (a–z, 0–9, `.`, `-`), không dùng shortname |
| SELinux | enforcing hoặc permissive; disabled không được hỗ trợ |
| Thời gian | Đồng bộ NTP (chrony); lệch giờ làm hỏng xác thực SSL |
| Locale | UTF-8 (ví dụ `en_US.utf-8`) |
| IPv6 | Không tắt IPv6 loopback (`ipv6.disable=1` hay `net.ipv6.conf.lo.disable_ipv6 = 1` làm hỏng cài đặt); không hỗ trợ dual-stack |
| Tài khoản | IdP bên ngoài không được tạo sẵn user `apache`, `foreman`, `foreman-proxy`, `postgres`, `pulp`, `puppet`, `redis`, `tomcat` |
| `/tmp` | Nếu tách riêng thì mount `exec` (puppetserver cần) |

Cần phần mềm bảo mật hay hardening của tổ chức: cài và kiểm thử Satellite xong, sao lưu, rồi mới thêm. Satellite không đăng ký vào chính nó.

Dung lượng (cài đặt / vận hành): `/var/log` 10 MB / 10 GB; `/var/lib/pgsql` 1 GB / 20 GB; `/usr` 10 GB; `/opt/puppetlabs` 500 MB; `/var/lib/pulp` 1 MB / 300 GB; `/var/lib/containers` 20 / 30 GB nếu bật Red Hat Lightspeed trong Satellite. Khuyến nghị đặt `/var` trên LVM, `/var/lib/pulp` và `/var/lib/pgsql` trên đĩa độ trễ thấp, không dùng GFS2, không dùng symlink cho `/var/lib/pulp` (NFS cho pulp phải khai báo SELinux context `pulpcore_var_lib_t` trong fstab). Tạo LV: xem [LVM](https://github.com/lovelybugf/kb-base-devops/blob/main/linux/storage-lvm.md).

## Bước 1. Chuẩn bị hệ điều hành

Chạy bằng root. Đặt hostname và kiểm tra phân giải thuận/nghịch:

```bash
hostnamectl set-hostname <SATELLITE_FQDN>
hostname -f
dig +short <SATELLITE_FQDN> A
dig +short -x <IP>
getent hosts $(hostname -f)
ping -c1 localhost
ping -c1 $(hostname -f)
```

- `getent hosts` phải trả FQDN mới: VM cài OS với hostname tạm thường còn dòng cũ trong `/etc/hosts` (lỗi đã gặp khi triển khai).
- Chứng chỉ do installer tự tạo gắn với FQDN lúc cài; đổi tên sau đó là cả một quy trình (Administering, chương 13).

Thời gian, SELinux và các điều kiện khác:

```bash
systemctl enable --now chronyd
chronyc sources
timedatectl
getenforce
umask
localectl status
ip -6 addr show lo
findmnt /tmp
df -hT /var /var/lib/pulp /var/lib/pgsql /var/log /usr
```

Mở cổng trên firewalld của Satellite (Installing connected, mục 2.1). Bộ đầy đủ theo tài liệu hãng dưới đây; chỉ mở dịch vụ thực dùng (không làm provisioning thì không cần dns/dhcp/tftp; ma trận cổng đầy đủ ở [Cổng mạng](../architecture/network-ports.md)):

```bash
firewall-cmd --add-port="8000/tcp" --add-port="9090/tcp"
firewall-cmd --add-service=dns --add-service=dhcp --add-service=tftp \
  --add-service=http --add-service=https --add-service=puppetmaster
firewall-cmd --runtime-to-permanent
firewall-cmd --list-all
```

## Bước 2. (Tuỳ chọn) HTTP proxy ra CDN

Proxy và gateway phải cho phép `subscription.rhsm.redhat.com:443` và `cdn.redhat.com:443` (thêm các host `console.redhat.com`, `cert.console.redhat.com`, `api.access.redhat.com`, `cert-api.access.redhat.com`, `connect.cloud.redhat.com` nếu dùng Red Hat Lightspeed). Khai báo trong `/etc/rhsm/rhsm.conf`: `proxy_hostname`, `proxy_port`, và `proxy_user`/`proxy_password` nếu proxy cần xác thực (mục 3.1, PDF tr. 17–18).

Kiểm tra không bị SSL inspection: issuer phải là CA công cộng, không phải CA nội bộ (thiết bị giải mã TLS làm hỏng sync CDN).

```bash
curl -v https://subscription.rhsm.redhat.com 2>&1 | grep -i issuer
```

## Bước 3. Đăng ký RHSM và bật repository

```bash
subscription-manager register
subscription-manager repos --disable "*"
subscription-manager repos \
  --enable=rhel-9-for-x86_64-baseos-rpms \
  --enable=rhel-9-for-x86_64-appstream-rpms \
  --enable=satellite-6.19-for-rhel-9-x86_64-rpms \
  --enable=satellite-maintenance-6.19-for-rhel-9-x86_64-rpms
dnf repolist enabled
```

Tài khoản bật Simple Content Access thì không cần `attach` pool (ghi chú RH403).

## Bước 4. Cài gói

Chạy trong `tmux` (installer kéo dài hàng chục phút; mất SSH thì xem `/var/log/foreman-installer/satellite.log`):

```bash
dnf install -y tmux
tmux new -s satinstall
dnf upgrade -y
dnf install -y satellite
dnf needs-restarting -r
```

Có kernel mới thì reboot trước khi chạy installer.

## Bước 5. Chạy satellite-installer

```bash
satellite-installer --scenario satellite --noop
satellite-installer --scenario satellite \
  --foreman-initial-organization "<ORG>" \
  --foreman-initial-location "<LOCATION>" \
  --foreman-initial-admin-username admin \
  --foreman-initial-admin-password '<PASSWORD>'
```

- Đặt tên Organization thật ngay từ đầu: label sinh ra từ tên này không đổi được về sau; bỏ trống thì có `Default Organization`.
- Không để mật khẩu admin trong history (`set +o history` trước lệnh) hoặc đổi mật khẩu ngay sau khi cài; lưu ở két mật khẩu ([bí mật trong tài liệu](../security/documentation-secrets.md)).
- Kết quả đúng: `Success!` kèm URL `https://<SATELLITE_FQDN>`. Lỗi: đọc log, sửa nguyên nhân rồi chạy lại cùng lệnh (installer chạy lại an toàn).
- Installer quản lý file cấu hình bằng Puppet: chạy lại sẽ ghi đè sửa tay. Dùng `--noop` để xem trước; file bị ghi đè khôi phục được bằng `puppet filebucket` (Phụ lục A, PDF tr. 47).
- Xem tham số: `satellite-installer --scenario satellite --help` (`--full-help` cho tham số nâng cao).
- PostgreSQL ngoài: thêm nhóm tham số `--katello-candlepin-db-*`, `--foreman-proxy-content-pulpcore-postgresql-*`, `--foreman-db-*` với `manage`/`manage-db` = `false` (mục 3.6, PDF tr. 20–21). Lightspeed trong Satellite không dùng được với DB ngoài.
- Nhiều host: áp profile `satellite-installer --tuning medium` (chương 4; bảng profile ở [Sizing và tuning](../architecture/sizing-and-tuning.md)).

## Bước 6. Cấu hình sau cài

1. Import manifest: Content → Subscriptions → Manage Manifest, hoặc `hammer subscription upload --file ~/manifest_file.zip --organization "<ORG>"` (mục 3.7). Chi tiết: [Manifest và subscription](../content/subscriptions-manifest.md).
2. Bật và sync repo **Red Hat Satellite Client 6** cho từng major RHEL của host (mục 5.3): cần cho `katello-host-tools`, REX, báo cáo gói. Xem [Repository và sync](../content/repositories-and-sync.md).
3. HTTP proxy cho sync nội dung (mục 5.6): unset biến `http_proxy`/`https_proxy` rồi:

```bash
hammer http-proxy create --name "<PROXY_NAME>" \
  --url http://<PROXY_FQDN>:8080 \
  --content-default-http-proxy true
```

   Proxy chạy cổng ngoài danh sách SELinux `http_cache_port_t` thì thêm: `semanage port -a -t http_cache_port_t -p tcp <PORT>`.
4. Các tuỳ chọn khác (mục 5.x, đều qua `satellite-installer`): đăng ký Satellite với Lightspeed `--register-with-insights`; REX pull-mqtt `--foreman-proxy-plugin-remote-execution-script-mode pull-mqtt` + mở `mqtt` (1883); BMC `--foreman-proxy-bmc true`; CNAME phụ `--certs-cname <ALT_FQDN> --certs-update-server`; email tại Administer → Settings → Email.
5. Chứng chỉ của CA doanh nghiệp: làm ngay trước khi cài Capsule và đăng ký host — [Chứng chỉ SSL](certificates.md).
6. Sao lưu lần đầu: [Sao lưu và khôi phục](../operations/backup-and-restore.md).

## Kiểm tra

```bash
satellite-maintain service list
satellite-maintain health check
hammer ping
hammer capsule list
```

Kết quả đúng: mọi dịch vụ `[OK]`; `hammer ping` báo `ok` cho database, candlepin, pulp3, foreman_tasks; `hammer capsule list` có Capsule tích hợp mang FQDN của Satellite; đăng nhập được `https://<SATELLITE_FQDN>`.

## Lỗi hay gặp

| Hiện tượng | Nguyên nhân | Xử lý |
|---|---|---|
| Installer lỗi liên quan hostname/DNS | Thiếu PTR, hostname không phải FQDN, còn tên tạm trong `/etc/hosts` | `hostname -f`, `dig -x`, `getent hosts`; sửa rồi chạy lại |
| Health check cảnh báo `sssd_kcm: Could not open file /var/log/sssd/sssd_kcm.log` | Thư mục log sssd chưa có | `mkdir -p /var/log/sssd; chmod 750 /var/log/sssd; systemctl restart sssd-kcm` (không ảnh hưởng dịch vụ Satellite) |
| Sync CDN lỗi `certificate verify failed` | Proxy làm SSL inspection, lệch giờ | Loại trừ domain Red Hat khỏi inspection; `chronyc tracking` |
| Cài đặt hỏng khi đã tắt IPv6 | `ipv6.disable=1` | Bật lại IPv6 loopback, chạy lại installer |
| Mất kết nối SSH giữa chừng | Phiên bị đóng | Gắn lại `tmux attach`, đọc `/var/log/foreman-installer/satellite.log` |

## Liên quan

- [Cài Satellite disconnected](install-satellite-disconnected.md) · [Cài Capsule Server](install-capsule.md) · [Chứng chỉ SSL](certificates.md) · [Nâng cấp và cập nhật](upgrade-satellite.md)
- [Cổng mạng](../architecture/network-ports.md) · [Thành phần và dịch vụ](../architecture/components-and-services.md) · [Health check](../operations/health-check-and-troubleshooting.md)

## Nguồn

- Red Hat Satellite 6.19, Installing Satellite Server in a connected network environment, chương 1 Planning (PDF tr. 8–13), chương 2 Preparing your environment (PDF tr. 14–16), chương 3 Installing Satellite Server (PDF tr. 17–23), chương 4 Tuning (PDF tr. 24–25), chương 5 Additional configuration (PDF tr. 26–46), Phụ lục A (PDF tr. 47) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- RH403, ghi chép quy trình cài đặt Satellite (SCA không cần attach pool, kiểm tra sau cài).
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: kiểm tra `getent hosts` vì tên tạm trong `/etc/hosts`, cảnh báo `sssd_kcm` khi health check, kiểm tra SSL inspection trước khi sync, chạy installer trong tmux.
