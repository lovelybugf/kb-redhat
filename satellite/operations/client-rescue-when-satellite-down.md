---
title: Cứu host RHEL khi Satellite hỏng
tags: [runbook, satellite, registration, capsule, rhel]
product: Red Hat Satellite 6.19, RHEL 9
source: "Ghi chép RH403 cứu hộ client khi Satellite chết (đã chạy thử); kinh nghiệm triển khai thực tế (ẩn danh); Red Hat Satellite 6.19 — Administering Red Hat Satellite, chương 4 (PDF tr. 22–32)"
reliability: internal-tested
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Satellite hỏng không làm host ngừng chạy; host đăng ký qua Capsule còn sống vẫn `dnf install` được, chỉ refresh, đăng ký mới và job từ Satellite là lỗi. Satellite chết lâu mà cần vá gấp thì dọn cấu hình Satellite trên host, tạm đăng ký về Red Hat CDN, khôi phục xong thì đăng ký lại.

## Ảnh hưởng khi Satellite ngừng

| Thao tác trên host | Satellite dừng, Capsule còn chạy | Cả Satellite và Capsule mất (hoặc host đăng ký thẳng vào Satellite) |
|---|---|---|
| Ứng dụng, dịch vụ đang chạy | Không ảnh hưởng | Không ảnh hưởng |
| `dnf install` / `dnf update` | **Được** (đã kiểm chứng: cài gói qua VIP Capsule thành công) | Lỗi: không tải được `repomd.xml`, lỗi HTTPS/SSL tới FQDN cũ |
| `subscription-manager refresh` | Lỗi HTTP 503 Service Unavailable (đã kiểm chứng) | Lỗi kết nối |
| Đăng ký host mới | Lỗi (yêu cầu đăng ký không được xử lý) | Lỗi |
| Job REX/Ansible, áp errata từ Web UI | Không làm được (Satellite điều phối) | Không làm được |
| Báo cáo trạng thái, errata, OpenSCAP về Satellite | Không cập nhật | Không cập nhật |

Lý do `dnf` qua Capsule vẫn chạy: Capsule phục vụ content từ Pulp của chính nó và xác thực host bằng chứng chỉ trên host, không cần Satellite trực tuyến. Vì vậy Capsule (nhất là cặp Capsule sau load balancer) là lớp đệm tốt nhất khi Satellite gặp sự cố.

## Chọn phương án

1. **Khôi phục Satellite** (ưu tiên): restore, snapshot hoặc chuyển sang máy passive theo [Khắc phục thảm hoạ](disaster-recovery-and-cloning.md). Host giữ nguyên đăng ký, không phải làm gì.
2. **Cài gói khẩn từ ISO RHEL** (phương án B): không đụng tới đăng ký; chỉ có gói ở mức minor của ISO, không có bản vá sau ngày phát hành.
3. **Tạm đăng ký về Red Hat CDN** (phương án C): có bản vá mới nhất; cần Internet ra Red Hat được phê duyệt, tài khoản/subscription Red Hat, và phải đăng ký lại về Satellite sau.

Phương án C bỏ qua Content View: host nhận phiên bản gói mới nhất trên CDN, có thể khác phiên bản đã kiểm thử trong CV. Chỉ cài đúng gói/errata cần thiết thay vì `dnf update` toàn bộ.

## Phương án B: repo cục bộ từ ISO RHEL

Chưa chạy trên host trong phiên kiểm chứng; cùng kỹ thuật đã dùng để cài gói cho chính Satellite khi offline. Chạy bằng root:

```bash
mkdir -p /mnt/rhel-iso
mount -o loop,ro <RHEL_ISO_PATH> /mnt/rhel-iso
cat > /etc/yum.repos.d/rhel-iso.repo << 'EOF'
[iso-baseos]
name=RHEL ISO BaseOS
baseurl=file:///mnt/rhel-iso/BaseOS
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
[iso-appstream]
name=RHEL ISO AppStream
baseurl=file:///mnt/rhel-iso/AppStream
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF
dnf --disablerepo='*' --enablerepo='iso-*' install <PKG>
```

Xong thì xoá file `.repo` và umount để host quay lại dùng repo của Satellite.

## Phương án C: chuyển tạm về Red Hat CDN

Các bước 1–5 đã chạy thành công trên lab RH403 (RHEL 9). Chạy bằng root trên host.

### 1. Huỷ đăng ký cũ, dọn cache

```bash
subscription-manager unregister
subscription-manager clean
dnf clean all
rm -rf /var/cache/dnf/*
```

Satellite đã chết thì `unregister` báo lỗi kết nối: bỏ qua, `clean` mới là bước xoá chứng chỉ và dữ liệu đăng ký trên máy.

### 2. Gỡ CA của Satellite cũ

```bash
rpm -qa | grep katello-ca && rpm -e $(rpm -qa | grep katello-ca)
rm -f /etc/pki/ca-trust/source/anchors/*katello* /etc/pki/ca-trust/source/anchors/*foreman*
rm -f /etc/rhsm/ca/katello* /etc/rhsm/ca/candlepin*
update-ca-trust extract
```

Gói `katello-ca-consumer-*` chỉ có ở host đăng ký theo cách cũ; gỡ gói này trả `rhsm.conf` về mặc định Red Hat. Không gõ tên gói kèm dấu `<` `>` (shell hiểu là chuyển hướng).

### 3. Trả cấu hình RHSM về Red Hat

```bash
subscription-manager config --rhsm.repo_ca_cert=/etc/rhsm/ca/redhat-uep.pem
```

Host đăng ký bằng global registration (cách mặc định ở Satellite 6.19) **không** có gói `katello-ca-consumer`; script đăng ký đã sửa `hostname`, `prefix`, `baseurl` trong `/etc/rhsm/rhsm.conf` và sao lưu bản gốc thành `rhsm.conf.bak` ở lần chạy đầu. Khi đó cần trả cả server về mặc định — một trong hai cách sau, **chưa kiểm chứng**:

```bash
# Cách 1: lấy lại bản gốc nếu còn
cp -a /etc/rhsm/rhsm.conf.bak /etc/rhsm/rhsm.conf
# Cách 2: đặt lại giá trị mặc định của Red Hat
subscription-manager config --server.hostname=subscription.rhsm.redhat.com \
  --server.port=443 --server.prefix=/subscription \
  --rhsm.baseurl=https://cdn.redhat.com \
  --rhsm.repo_ca_cert=/etc/rhsm/ca/redhat-uep.pem
```

Host ra Internet qua proxy thì đặt `--server.proxy_hostname`/`--server.proxy_port` theo proxy của tổ chức; ngược lại xoá proxy cũ trỏ tới Satellite/Capsule.

### 4. Kiểm tra

```bash
subscription-manager config --list | grep -E 'hostname|repo_ca_cert'
```

Mong đợi: `hostname = [subscription.rhsm.redhat.com]`, `proxy_hostname = []` (hoặc proxy của tổ chức), `repo_ca_cert` trỏ `redhat-uep.pem`.

### 5. Đăng ký về CDN và bật repo

```bash
subscription-manager register --username=<USER>
# hoặc dùng activation key tạo trên Red Hat Hybrid Cloud Console:
# subscription-manager register --org=<ORG_ID> --activationkey=<ACTIVATION_KEY>
subscription-manager repos --enable=rhel-9-for-x86_64-baseos-rpms \
  --enable=rhel-9-for-x86_64-appstream-rpms
dnf clean all
dnf install -y <PKG>
```

Không ghi `--password` trên dòng lệnh: bỏ tuỳ chọn này để lệnh tự hỏi, tránh lộ mật khẩu trong lịch sử shell.

### Lỗi hay gặp

| Lỗi | Nguyên nhân | Xử lý |
|---|---|---|
| `-bash: ...: No such file or directory` khi gỡ gói | Gõ tên gói có `<` `>` | Dùng `rpm -e $(rpm -qa \| grep katello-ca)` |
| `Curl error (60): SSL certificate problem: self-signed certificate` | CA Satellite cũ còn trong CA trust hệ thống | Xoá anchor `*katello*`, `update-ca-trust extract` |
| `Curl error (77): error setting certificate file: /etc/rhsm/ca/katello-server-ca.pem` | `repo_ca_cert` vẫn trỏ CA Satellite đã xoá | `subscription-manager config --rhsm.repo_ca_cert=/etc/rhsm/ca/redhat-uep.pem` |
| `register` vẫn gọi tới FQDN Satellite cũ | `server.hostname` chưa trả về mặc định (host dùng global registration) | Làm bước 3 cách 1 hoặc 2 |

### Làm hàng loạt

Gom bước 1–3 vào một script, đẩy bằng công cụ sẵn có (Ansible, SSH vòng lặp) vì REX của Satellite không dùng được; bước đăng ký chạy riêng để không đưa thông tin đăng nhập vào script:

```bash
#!/bin/bash
subscription-manager unregister 2>/dev/null
subscription-manager clean
PKG=$(rpm -qa | grep katello-ca); [ -n "$PKG" ] && rpm -e "$PKG"
rm -f /etc/pki/ca-trust/source/anchors/*katello* /etc/pki/ca-trust/source/anchors/*foreman*
rm -f /etc/rhsm/ca/katello* /etc/rhsm/ca/candlepin*
update-ca-trust extract
subscription-manager config --rhsm.repo_ca_cert=/etc/rhsm/ca/redhat-uep.pem
dnf clean all; rm -rf /var/cache/dnf/*
echo "San sang dang ky ve Red Hat CDN"
```

## Quay lại Satellite sau khi khôi phục

1. Trên host: `subscription-manager unregister` (huỷ đăng ký trên CDN, trả lại subscription), `subscription-manager clean`; gỡ repo ISO nếu đã thêm.
2. Bản ghi host cũ vẫn còn trên Satellite: bật **Force** trong form đăng ký, hoặc `hammer host delete --name "<HOST_FQDN>"` trước.
3. Chạy lại lệnh global registration sinh từ Hosts > Register Host với `<ORG>`, `<ACTIVATION_KEY>` cũ, rồi `subscription-manager identity` và `dnf repolist`.
4. So phiên bản gói đã cài từ CDN với CV hiện tại; nếu host cao hơn CV thì cân nhắc publish/promote CV mới, tránh host "vượt" lifecycle.

Chi tiết các bước đăng ký lại: [Huỷ đăng ký và đăng ký lại host](../hosts/host-reregistration.md).

## Liên quan

- [Huỷ đăng ký và đăng ký lại host](../hosts/host-reregistration.md)
- [Đăng ký host](../hosts/host-registration.md)
- [Khắc phục thảm hoạ, clone và đổi tên Satellite](disaster-recovery-and-cloning.md)
- [Sao lưu và khôi phục](backup-and-restore.md)
- [Capsule sau load balancer](../installation/capsule-load-balancer.md)
- [RHEL 10 GPG check FAILED](../../rhel/rhel10-gpg-release-key-4.md)

## Nguồn

- Ghi chép khoá RH403: quy trình 5 bước chuyển host từ Satellite về Red Hat CDN và bảng lỗi `Curl error (60)/(77)`, đã chạy thành công trên lab RHEL 9.
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: dừng dịch vụ Satellite, host vẫn `dnf install` qua VIP Capsule; `subscription-manager refresh` lỗi HTTP 503; đăng ký host mới lỗi; khởi động lại thì dịch vụ tự chạy và host refresh lại bình thường.
- Red Hat Satellite 6.19, Administering Red Hat Satellite, chương 4 Preparing for disaster recovery (PDF tr. 22–32) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Phần trả `rhsm.conf` về mặc định cho host dùng global registration và phương án repo ISO trên host: chưa kiểm chứng.
