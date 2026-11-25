---
title: Đăng ký host lên Satellite bằng global registration
tags: [howto, satellite, registration, capsule, remote-execution, hammer]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Managing hosts, chương 6, mục 19.2.2 (PDF tr. 35–58, 119–120); Installing Capsule Server, mục 3.1–3.2 (PDF tr. 21–22); Administering, Phụ lục F.7 (PDF tr. 140–144); RH403 chương 3; kinh nghiệm triển khai (ẩn danh)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Hosts → Register Host (hoặc `hammer host-registration generate-command`) sinh một lệnh đăng ký dạng curl có token JWT; chạy lệnh đó bằng root trên host để đăng ký qua Satellite hoặc Capsule với Activation Key đã chọn. Cùng lệnh đó có thể bật REX, Lightspeed, cài/cập nhật gói; bootstrap.py và gói katello-ca-consumer đã deprecated.

## Chọn phương thức

| Phương thức | Trạng thái | Ghi chú |
|---|---|---|
| Global registration (lệnh curl/wget sinh từ Satellite) | Mặc định, khuyến nghị | Dùng template *Global Registration* + *Linux host_init_config default*; chạy được trên host mới cài lẫn host từng đăng ký nơi khác |
| `bootstrap.py` | Deprecated | Không chạy qua Capsule (reverse proxy trên Capsule tắt mặc định); chỉ còn cho host cũ |
| Gói `katello-ca-consumer-latest.noarch.rpm` + `subscription-manager register` | Deprecated | Giáo trình RH403 bản cũ còn dùng; triển khai mới không dùng |

Host đang đăng ký nơi khác hoặc cần đăng ký lại: xem [Huỷ đăng ký và đăng ký lại host](host-reregistration.md). Activation Key (AK) chỉ áp dụng một lần lúc đăng ký: xem [Activation Key chỉ áp dụng lúc đăng ký](activation-key-registration-only.md).

## Chuẩn bị

**Phía Satellite/Capsule**

- User sinh lệnh có role **Register hosts** (hoặc quyền tương đương). Token JWT mang quyền của user này suốt thời hạn của nó, vì vậy không xoá, khoá hay đổi quyền user khi token còn hạn.
- AK trỏ tới Content View/Lifecycle Environment có repo **Red Hat Satellite Client 6** đúng bản RHEL của host (cần cho pull client, Tracer, OpenVox...). Muốn cài Lightspeed lúc đăng ký thì CV phải có thêm BaseOS + AppStream.
- Đăng ký qua Capsule: Capsule có tính năng **Registration** và **Templates** (Infrastructure → Capsules), đã được gán LE của AK và đã sync; IP Capsule nằm trong danh sách trusted proxies của Satellite (xem mục *Đăng ký qua Capsule*).

**Phía host** (RHEL 8/9/10; RHEL 7 cần ELS Add-On)

- Có `curl` (hoặc `wget`), chạy lệnh bằng root.
- Đồng bộ giờ cùng nguồn NTP với Satellite/Capsule; lệch giờ làm hỏng kiểm tra chứng chỉ SSL.
- Tin CA của Satellite trước khi đăng ký (thay cho tuỳ chọn Insecure). Chạy bằng root trên host:

```bash
systemctl enable --now chronyd
chronyc tracking
curl -o /etc/pki/ca-trust/source/anchors/satellite-ca.crt http://<SATELLITE_FQDN>/pub/katello-server-ca.crt
openssl x509 -noout -subject -fingerprint -sha256 -in /etc/pki/ca-trust/source/anchors/satellite-ca.crt
update-ca-trust extract
```

Đối chiếu dấu vân tay SHA256 với bản gốc trên Satellite (`/var/www/html/pub/katello-server-ca.crt`) trước khi tin. Tài liệu hãng dùng cách chép file CA (đường dẫn ghi ở Administer → Settings → Authentication → *SSL CA file*) sang `/etc/pki/ca-trust/source/anchors/` bằng `scp`.

## Global parameter quyết định hành vi mặc định

Configure → Global Parameters; có thể ghi đè ở organization, location, host group, host (xem [Host group và tham số](host-groups-and-parameters.md)).

| Tham số | Mặc định | Tác dụng lúc đăng ký |
|---|---|---|
| `host_registration_insights` | `true` | Cài và đăng ký insights-client (Lightspeed); kiểu phải là boolean |
| `host_registration_insights_inventory` | `true` | Đưa host vào báo cáo inventory gửi lên Hybrid Cloud Console |
| `host_registration_remote_execution` | `true` | Cài khoá SSH để chạy REX |
| `host_packages` | rỗng | Danh sách gói cài sau đăng ký |
| `remote_execution_ssh_user`, `remote_execution_create_user`, `remote_execution_effective_user_method`, `remote_execution_ssh_keys` | — | Dùng trong snippet `remote_execution_ssh_keys` |

Các tham số này chỉ tác động lúc đăng ký; tắt Lightspeed cho cả hệ thống: [Tắt Red Hat Lightspeed](disable-lightspeed.md).

## Sinh lệnh trên Web UI

1. Chọn đúng Organization/Location ở thanh trên → **Hosts → Register Host**.
2. Điền các trường cần (bảng dưới) → **Generate** → copy lệnh.
3. Chạy lệnh bằng root trên host. Nếu có chọn host group gắn Ansible role, các role chạy ngay sau khi đăng ký.

| Trường (tab) | Ý nghĩa | Tuỳ chọn hammer tương ứng |
|---|---|---|
| Organization, Location (General) | Nơi tạo bản ghi host | `--organization`, `--location` |
| Host Group (General) | Điền sẵn OS, Activation Keys, LE từ host group | `--hostgroup` |
| Operating System (General) | Gán OS cho bản ghi host | `--operatingsystem` |
| Capsule (General) | Host đăng ký và lấy nội dung qua Capsule này; Capsule sau load balancer được ưu tiên hơn Capsule chọn ở đây | `--smart-proxy`, `--smart-proxy-id` |
| Insecure (General) | Lần gọi đầu không kiểm CA, host tự tải CA về; kẻ đứng giữa có thể lấy CA, đọc JWT và (nếu bật REX) dùng khoá SSH vào host | `--insecure true` |
| Download utility (General) | `curl` (mặc định) hoặc `wget` | xem `--help` |
| Activation Keys (General) | Một hoặc nhiều AK; khi xung đột thiết lập, key đứng sau thắng | `--activation-keys` |
| Setup REX (Advanced) | Cài khoá SSH cho REX kiểu push (SSH); mặc định kế thừa global parameter | `--setup-remote-execution` |
| REX pull mode (Advanced) | Cài pull client cho Capsule chạy `pull-mqtt` | `--setup-remote-execution-pull` |
| Remote Execution Interface (Advanced) | Card mạng dùng cho REX | `--remote-execution-interface` |
| Setup Lightspeed/Insights (Advanced) | Cài và đăng ký insights-client | `--setup-insights` |
| Install packages / Update packages (Advanced) | Cài thêm gói; cập nhật toàn bộ gói sau đăng ký | `--packages`, `--update-packages` |
| Repositories, Repository GPG key URL (Advanced) | Repo thêm vào **trước** khi đăng ký; bỏ trống nếu AK đã cung cấp repo | xem `--help` |
| Token lifetime (hours) (Advanced) | Hạn JWT = thời gian lệnh còn dùng được; mặc định 4 giờ | `--jwt-expiration` |
| Ignore errors (Advanced) | Bỏ qua lỗi của subscription-manager | `--ignore-subman-errors` |
| Force (Advanced) | Gỡ gói katello-ca-consumer cũ và gọi `subscription-manager register --force` để ghi đè đăng ký đang có | `--force true` |
| Set up container registry certs (Advanced) | Pull container/Flatpak bằng chứng chỉ của host, không cần user/mật khẩu | — |

Đã dùng thực tế: `--organization`, `--location`, `--activation-keys`, `--smart-proxy`/`--smart-proxy-id`, `--setup-insights`, `--setup-remote-execution`; `--insecure` có trong tài liệu hãng. Các tuỳ chọn còn lại lấy theo tên trường, kiểm `hammer host-registration generate-command --help` trước khi dùng.

## Sinh lệnh bằng hammer và chạy từ xa

```bash
hammer host-registration generate-command \
  --organization "<ORG>" \
  --location "<LOCATION>" \
  --activation-keys "<ACTIVATION_KEY>" \
  --smart-proxy "<CAPSULE_FQDN>" \
  --setup-remote-execution false \
  --setup-insights false
```

Lệnh sinh ra có dạng `set -o pipefail && curl --silent --show-error 'https://<CAPSULE_FQDN>:9090/register?...' --header 'Authorization: Bearer <TOKEN>' | bash` (qua Satellite trực tiếp thì là `https://<SATELLITE_FQDN>/register?...`). Không dán lệnh, token vào tài liệu hay ảnh chụp: xem [bí mật trong tài liệu](../security/documentation-secrets.md).

Chạy thẳng trên host từ máy có hammer: đưa lệnh vào **stdin** của ssh.

```bash
hammer host-registration generate-command \
  --organization "<ORG>" --location "<LOCATION>" \
  --activation-keys "<ACTIVATION_KEY>" --smart-proxy "<CAPSULE_FQDN>" \
  | ssh <USER>@<HOST_FQDN> 'sudo bash -s'
```

Nhét lệnh vào trong chuỗi `ssh host "<lệnh>"` làm shell cắt mất token, host nhận 401. Cách khác: API `POST /api/registration_commands`, hoặc module Ansible `redhat.satellite.registration_command`.

## Đăng ký qua Capsule

- Host gọi Capsule cổng 9090 (đăng ký) và 443 (nội dung): xem [cổng mạng](../architecture/network-ports.md).
- Satellite chỉ tin header `X-Forwarded-For` từ localhost; thêm IP Capsule vào trusted proxies. Lệnh **ghi đè** cả danh sách, phải liệt kê lại các mục cũ và giữ hai mục localhost. Chạy bằng root trên Satellite:

```bash
satellite-installer \
  --foreman-trusted-proxies "127.0.0.1/8" \
  --foreman-trusted-proxies "::1" \
  --foreman-trusted-proxies "<CAPSULE_IP>"
satellite-installer --full-help | grep -A 2 "trusted-proxies"
```

- Nhiều Capsule sau load balancer: xem [Capsule sau load balancer](../installation/capsule-load-balancer.md).

## Remote execution kiểu pull

Khi Capsule chạy `pull-mqtt`, host không cần mở cổng SSH vào; host tự kết nối Capsule cổng 1883 (MQTT) và 9090. Chọn **REX pull mode** lúc đăng ký, hoặc sau đăng ký cài gói (host phải bật repo Satellite Client 6 và AppStream). Chạy bằng root trên host:

```bash
dnf install katello-pull-transport-migrate
rpm -q yggdrasil
systemctl status yggdrasil com.redhat.Yggdrasil1.Worker1.foreman
```

Với yggdrasil 0.2.z trở xuống, service tên `yggdrasild`. Trên Satellite đặt `remote_execution_prefer_registered_through_proxy = true` để job đi qua đúng Capsule host đã đăng ký. Chi tiết: [Remote execution](remote-execution.md).

## Kiểm tra sau đăng ký

Trên host:

```bash
subscription-manager identity
subscription-manager config --list | grep hostname
subscription-manager repos --list-enabled
dnf repolist
```

`hostname` trong cấu hình rhsm phải là Capsule (nếu đăng ký qua Capsule); `dnf repolist` phải có các repo AK cấp, gồm Satellite Client 6. Trên Satellite: **Hosts → All Hosts** → host: *Registered through* đúng Capsule, đúng Content View/LE, tab **Content → Repository sets** đúng trạng thái.

## Lỗi hay gặp

| Triệu chứng | Nguyên nhân | Cách xử lý |
|---|---|---|
| `401 Unauthorized` khi chạy lệnh | Token hết hạn/bị vô hiệu, hoặc lệnh bị cắt khi truyền qua ssh | Sinh lệnh mới; truyền qua stdin như trên |
| `SSL certificate problem` | Host chưa tin CA, hoặc lệch giờ | Cài CA, đồng bộ NTP; tránh dùng Insecure |
| `This system is already registered` | Host còn đăng ký cũ | Bật Force, hoặc làm theo [đăng ký lại](host-reregistration.md) |
| Đăng ký qua Capsule lỗi hoặc host nhận IP của Capsule | Thiếu trusted proxies | Thêm IP Capsule như mục trên |
| `dnf repolist` trống sau đăng ký | CV/LE của AK chưa có repo, Capsule chưa gán LE hoặc chưa sync | Kiểm tra CV version và Capsule sync; `subscription-manager refresh` |
| Hai máy clone đè lên cùng một bản ghi host | Trùng DMI UUID | Thêm UUID vào setting *Host Duplicate DMI UUIDs* (Administer → Settings → Content) rồi đăng ký lại |
| Tên host trên Satellite không phải FQDN | Máy không trả FQDN | Setting *Subscription manager name registration fact* chỉ định fact làm tên host |

Host mới đăng ký không chỉ định Location sẽ rơi vào setting *Default Location subscribed hosts* (tab Content).

## Bảo mật token đăng ký

- Phạm vi JWT chỉ là các endpoint đăng ký, nhưng mang quyền của user sinh lệnh; đặt Token lifetime ngắn.
- Vô hiệu token của mình: user menu → **My Account → Registration Tokens → Invalidate JWTs**; của user khác hoặc tất cả: **Administer → Users**. Hammer: `hammer user registration-token invalidate --user-id <USER_ID>`, nhiều user: `hammer user registration-token invalidate-multiple --search "<SEARCH>"`.
- Tạm khoá (disable) user làm token tạm mất hiệu lực; mở lại thì token dùng tiếp được.

## Tuỳ biến quá trình đăng ký

- Chèn bước riêng: tạo snippet tên đúng `before_registration` hoặc `after_registration` (Hosts → Templates → Provisioning Templates, type Snippet, gán Organization/Location); template Global Registration tự nạp.
- Sửa sâu hơn: clone *Global Registration* / *Linux host_init_config default* (template gốc bị khoá, bản clone không được Red Hat cập nhật), rồi trỏ setting *Default Global registration template* và *Default 'Host initial configuration' template* (Administer → Settings → Provisioning) sang bản clone.
- Lệnh sinh ra dùng ID; có thể sửa tham số URL sang tên (URL-encode): `organization=My%20Org`, host group lồng nhau `hostgroup=Parent%2FChild`.

## Liên quan

- [Activation Key chỉ áp dụng lúc đăng ký](activation-key-registration-only.md)
- [Huỷ đăng ký và đăng ký lại host](host-reregistration.md)
- [Host group, tham số, host collection, system purpose](host-groups-and-parameters.md)
- [Huỷ đăng ký và xoá host khỏi Satellite](host-deletion-and-unregister.md)
- [Remote execution](remote-execution.md) · [Tắt Lightspeed](disable-lightspeed.md)
- [Chứng chỉ Satellite/Capsule](../installation/certificates.md) · [Capsule sau load balancer](../installation/capsule-load-balancer.md)

## Nguồn

- Red Hat Satellite 6.19, Managing hosts, chương 6 Registering hosts to Satellite (PDF tr. 35–58: 6.2 global registration, 6.3 invalidating tokens, 6.4 bootstrap, 6.5 katello-ca-consumer) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing hosts, mục 19.2.2 Configuring a host to use the pull client (PDF tr. 119–120) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Installing Capsule Server, mục 3.1 (trusted proxies), 3.2 (pull-mqtt) (PDF tr. 21–22) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Administering Red Hat Satellite, Phụ lục F.7 Content management settings (PDF tr. 140–144) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Khoá RH403 (Red Hat Satellite 6.15), chương 3 Register Hosts — cách giải thích các phương thức đăng ký.
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: đăng ký qua Capsule bằng lệnh sinh từ Web UI/hammer với Setup REX = No, Lightspeed = No, không bật Insecure (host tải CA từ `/pub` và đối chiếu SHA256); token mặc định 4 giờ; truyền lệnh qua stdin của ssh, nhét lệnh vào chuỗi thì mất token (401).
