---
title: Cài Satellite Server 6.19 không có Internet (disconnected)
tags: [runbook, satellite, installation, rhel]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Installing Satellite Server in a disconnected network environment, chương 2, 3, 5 và phụ lục A–C (PDF tr. 13–24, 27–49); kinh nghiệm triển khai thực tế (ẩn danh)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Satellite không có Internet được cài từ hai ISO (RHEL 9 Binary DVD và Satellite 6.19 Binary DVD) bằng script `install_packages` thay cho đăng ký RHSM. Sau đó chạy `satellite-installer` như bản connected, import manifest xuất từ Customer Portal và chọn nguồn nội dung: Custom CDN, Export Sync hoặc Network Sync.

## Hai nghĩa của "disconnected"

| Tình huống | Cách cài gói Satellite | Nguồn nội dung cho host |
|---|---|---|
| Máy Satellite không đăng ký RHSM nhưng được mở Internet có kiểm soát (allow-list tới CDN) | ISO + `install_packages` | Red Hat CDN, Subscription Connection = Yes |
| Hoàn toàn air-gapped | ISO + `install_packages` | Custom CDN nội bộ, hoặc Satellite thượng nguồn (Export Sync / Network Sync), Subscription Connection = No |

Trong triển khai thực tế, mô hình thứ nhất khá phổ biến: Satellite là máy duy nhất ra Internet, Capsule và host không bao giờ ra Internet. Phần chuẩn bị hệ điều hành (FQDN, DNS thuận/nghịch, NTP, SELinux, phân vùng, firewall) giống hệt bản connected: xem [Cài Satellite connected](install-satellite-connected.md), Bước 1. Không có mục HTTP proxy và đăng ký RHSM.

ISO có thể không chứa bản vá mới nhất; Red Hat chỉ khuyến nghị cách này khi máy thật sự không ra được Internet (chương 3, PDF tr. 16).

## Bước 1. Tải ISO (trên máy có Internet)

Customer Portal → Downloads → Red Hat Enterprise Linux → bản minor mới nhất, x86_64 → **Binary DVD**; tương tự Red Hat Satellite → 6.19 → **Binary DVD** (mục 3.1). Chép ISO sang máy Satellite (hoặc gắn qua ổ CD ảo của VM).

## Bước 2. Tạo repository cục bộ từ ISO

Chạy bằng root:

```bash
mkdir -p /media/rhel /media/sat6
mount -o loop <RHEL_ISO> /media/rhel
mount -o loop <SATELLITE_ISO> /media/sat6
cat > /etc/yum.repos.d/offline.repo << 'EOF'
[RHEL-BaseOS]
name=Red Hat Enterprise Linux BaseOS
gpgcheck=0
baseurl=file:///media/rhel/BaseOS/
enabled=1

[RHEL-AppStream]
name=Red Hat Enterprise Linux AppStream
gpgcheck=0
baseurl=file:///media/rhel/AppStream/
enabled=1
EOF
dnf repolist
findmnt -t iso9660
```

ISO gắn qua ổ CD ảo thì thay `mount -o loop <ISO>` bằng `mount /dev/sr0 /media/rhel` và `mount /dev/sr1 /media/sat6`. Tài liệu hãng chép `media.repo` từ ISO rồi thêm `baseurl` (mục 3.2); kết quả tương đương.

Mọi lệnh `dnf` sẽ in "This system is not registered…": bình thường với cài đặt disconnected, **không** chạy `subscription-manager register` trên Satellite.

## Bước 3. Cài gói Satellite

```bash
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
dnf upgrade -y
dnf install -y tmux bind-utils
cd /media/sat6/
./install_packages
```

Kết quả đúng: `Install is complete. Please run satellite-installer --scenario satellite.`

Lỗi phụ thuộc gói (mục 3.4): trên máy có Internet vào Customer Portal → Downloads → sản phẩm chứa gói → tab **Packages** → tải đúng phiên bản; chép sang, `dnf install ./<PACKAGE>.rpm`, rồi chạy lại `./install_packages` đến khi báo hoàn tất. Nguyên nhân thường gặp: OS chưa cập nhật từ ISO RHEL mới nhất.

## Bước 4. Chạy satellite-installer và dọn repo tạm

Lệnh installer, lưu ý về Organization label, mật khẩu, `--noop`, DB ngoài: giống bản connected (Bước 5 của [Cài Satellite connected](install-satellite-connected.md)). Sau khi `Success!`:

```bash
umount /media/sat6
umount /media/rhel
mv /etc/yum.repos.d/offline.repo /etc/yum.repos.d/offline.repo.disabled
dnf repolist
```

Giữ lại file `.disabled` để dùng khi cần cài thêm gói từ ISO. Sau bước này `dnf repolist` chỉ còn hai repo cục bộ do `install_packages` tạo (ghi nhận khi triển khai: `Satellite-local`, `Maintenance-local`). Để repo ISO trỏ tới đường dẫn đã umount sẽ làm `dnf`/`satellite-maintain` báo lỗi về sau.

## Bước 5. Subscription Connection và manifest

Air-gapped hoàn toàn: tắt kết nối tới Red Hat Portal (mục 3.7–3.8). Hệ quả: không refresh manifest trực tiếp được, phải xuất manifest mới từ Portal và upload lại.

```bash
hammer settings set --name subscription_connection_enabled --value false
```

Web UI: Administer → Settings → Content → **Subscription Connection Enabled** = No. Mô hình có allow-list Internet thì để Yes.

Manifest (mục 3.9): trên máy có Internet tạo subscription allocation loại Satellite 6.19, thêm subscription RHEL và Satellite Infrastructure (gồm Capsule), xuất file `.zip`, chép sang Satellite rồi:

```bash
hammer subscription upload --file ~/manifest_file.zip --organization "<ORG>"
```

SCA bật/tắt theo Organization, không theo manifest. Chi tiết: [Manifest và subscription](../content/subscriptions-manifest.md).

## Bước 6. Chọn nguồn nội dung (CDN Configuration)

Content → Subscriptions → Manage Manifest → tab **CDN Configuration**, hoặc `hammer organization configure-cdn`:

| Kiểu | Khi nào | Lệnh |
|---|---|---|
| Custom CDN | Có web server nội bộ mirror đúng cấu trúc thư mục của Red Hat CDN | `hammer organization configure-cdn --name="<ORG>" --type=custom_cdn --url https://<CDN_FQDN> --ssl-ca-credential-id "<CA_CREDENTIAL_ID>"` |
| Export Sync | Satellite thượng nguồn và hạ nguồn air-gapped, chuyển nội dung bằng file | `hammer organization configure-cdn --name="<ORG>" --type=export_sync` |
| Network Sync | Satellite hạ nguồn kéo nội dung qua HTTPS từ Satellite thượng nguồn | `hammer organization configure-cdn --name="<ORG>" --type=network_sync --url https://<UPSTREAM_FQDN> --username <USER> --password <PASSWORD> --ssl-ca-credential-id "<CA_CREDENTIAL_ID>" --upstream-organization-label "<UPSTREAM_ORG_LABEL>"` |

- Custom CDN dùng HTTPS: upload CA của CDN thành content credential trước (mục 5.2–5.3).
- Network Sync (mục 5.4.4–5.4.5): cùng một manifest ở hai bên; CA thượng nguồn lấy từ `http://<UPSTREAM_FQDN>/pub/katello-server-ca.crt`; user thượng nguồn cần quyền xem organization, product, LE, CV và `export_content` (hoặc dùng personal access token thay mật khẩu). Có thể chỉ định thêm `--upstream-lifecycle-environment-label` (mặc định `Library`) và `--upstream-content-view-label` (mặc định `Default_Organization_View`).
- ISS Export Sync chỉ chạy khi các Satellite cùng phiên bản; Network Sync chạy chéo phiên bản (chương 1).
- Quy trình export/import chi tiết: [Inter-Satellite Sync](../content/inter-satellite-sync.md).

Sau đó bật repo và sync như bình thường: [Repository và sync](../content/repositories-and-sync.md).

## Red Hat Lightspeed trong Satellite khi không có Internet

Tóm tắt mục 5.1 (PDF tr. 27–30): podman phải dùng backend netavark, không dùng được với DB ngoài. Hai cách nạp container image: chạy `/media/sat6/setup_containers` từ ISO Satellite, hoặc `podman save` trên máy có Internet rồi `podman load` trên Satellite; sau đó `satellite-installer --enable-iop`. Tệp `cvemap.xml` tải từ `https://security.access.redhat.com/data/meta/v1/cvemap.xml` trên máy có Internet, chép vào `/var/lib/foreman/` (Satellite tự công bố lại). Không dùng thì xem [Tắt Lightspeed](../hosts/disable-lightspeed.md).

## Kiểm tra

```bash
satellite-maintain service list
satellite-maintain health check
hammer ping
hammer organization info --name "<ORG>"
```

Thêm: Content → Subscriptions hiển thị subscription của manifest; tab CDN Configuration đúng kiểu đã chọn; sync thử một repo nhỏ (ví dụ Satellite Client 6) thành công.

## Lỗi hay gặp

| Hiện tượng | Nguyên nhân | Xử lý |
|---|---|---|
| `install_packages` báo lỗi dependency | ISO thiếu gói hoặc OS chưa cập nhật từ ISO RHEL mới | Tải gói lẻ từ Customer Portal, cài cục bộ, chạy lại (mục 3.4) |
| `dnf` lỗi sau khi cài xong | Repo ISO vẫn bật nhưng ISO đã umount | Đổi tên `offline.repo` thành `.disabled` |
| Không refresh được manifest | Subscription Connection = No | Xuất manifest mới trên Portal, upload lại; không xoá manifest (mất entitlement của host) |
| Network Sync lỗi SSL | Chưa import CA thượng nguồn hoặc chọn sai content credential | Import `katello-server-ca.crt` của thượng nguồn, chọn đúng trong CDN Configuration |
| Hostname/DNS, `sssd_kcm`, lệch giờ | Như bản connected | Xem bảng lỗi ở [Cài Satellite connected](install-satellite-connected.md) |

## Phụ lục: chuyển về tải nội dung từ Red Hat CDN

Khi môi trường chuyển sang có Internet (Phụ lục B–C, PDF tr. 48–49):

- Web UI: Content → Subscriptions → Manage Manifest → CDN Configuration → **Red Hat CDN**, URL `https://cdn.redhat.com` → Update. Lần sync kế tiếp sẽ lấy từ CDN.
- Hammer:

```bash
hammer organization configure-cdn --name="<ORG>" --type=redhat_cdn
```

Nếu trước đó đã tắt Subscription Connection, cân nhắc bật lại để refresh manifest trực tiếp (bước này suy ra từ mục 3.7, phụ lục không ghi). Cập nhật bản vá Satellite về sau: [Nâng cấp và cập nhật](upgrade-satellite.md).

## Liên quan

- [Cài Satellite connected](install-satellite-connected.md) · [Cài Capsule Server](install-capsule.md) · [Chứng chỉ SSL](certificates.md) · [Nâng cấp và cập nhật](upgrade-satellite.md)
- [Lập kế hoạch triển khai](../architecture/deployment-planning.md) · [Download policy](../content/download-policy.md)

## Nguồn

- Red Hat Satellite 6.19, Installing Satellite Server in a disconnected network environment, chương 2 Preparing your environment (PDF tr. 13–15), chương 3 Installing Satellite Server (PDF tr. 16–24), chương 5 Additional configuration, mục 5.1–5.4 (PDF tr. 27–35), Phụ lục A–C (PDF tr. 47–49) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: mô hình "cài từ ISO, Satellite ra CDN qua allow-list"; tắt repo ISO sau cài; thông báo "not registered" của dnf là bình thường; repo cục bộ do `install_packages` tạo.
