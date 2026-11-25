---
title: Thành phần, dịch vụ và mô hình kết nối của Satellite
tags: [concept, satellite, capsule, pulp, monitoring]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Overview, concepts, and deployment considerations, chương 1, 5, 7, 8, 11.1, phụ lục A (PDF tr. 8, 19–21, 26–30, 42–43, 65–66); Administering Red Hat Satellite, phụ lục A (PDF tr. 118); RH403 chương 1"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Satellite Server là Foreman cộng plugin Katello (Candlepin lo subscription, Pulp lo content) trên nền PostgreSQL, Redis, httpd; Capsule là Smart Proxy (`foreman-proxy`) cộng Pulp, đặt gần host. Dịch vụ quản lý bằng `satellite-maintain service`; content vào từ Red Hat CDN (connected) hoặc Satellite khác qua ISS (disconnected).

## Luồng content bốn lớp

```text
Nguồn bên ngoài: Red Hat CDN, repo Yum tuỳ chỉnh, registry container, Git, SCAP, Ansible collection
        │  (Satellite kéo về, HTTPS)
        ▼
Satellite Server: Library, Content View, Lifecycle Environment, subscription, RBAC, Web UI/API
        │  (Capsule kéo content của các LE được gán)
        ▼
Capsule Server(s): bản sao content + dịch vụ cục bộ (đăng ký, DHCP/DNS/TFTP, REX, OpenSCAP)
        │
        ▼
Host (vật lý, ảo, cloud): đăng ký và lấy content từ Capsule của mình hoặc trực tiếp từ Satellite
```

- Mọi kế hoạch vòng đời content (sync, publish, promote) làm trên Satellite; Capsule chỉ phân phối.
- Satellite luôn có **Capsule tích hợp**, nên host có thể đăng ký thẳng vào Satellite. Capsule rời trước hết là một host đăng ký vào Satellite, sau đó mới được cài thành Capsule.
- Capsule không bao giờ đăng ký vào Red Hat CDN và phải cùng phiên bản với Satellite (Capsule 6.19 không đăng ký được vào Satellite 6.18).
- Satellite không thể tự đăng ký vào chính nó; hệ điều hành của Satellite cập nhật từ CDN (connected) hoặc media offline (disconnected).

## Các thành phần chính

| Thành phần | Vai trò | Có trên |
|---|---|---|
| Foreman | Quản lý vòng đời host: provisioning, cấu hình, giám sát; cung cấp Web UI và REST API (Hammer, Ansible collection đều gọi API này) | Satellite |
| Katello | Plugin của Foreman cho content, subscription, repository, CV/LE/Activation Key | Satellite |
| Candlepin | Dịch vụ subscription: manifest, SCA, chứng chỉ định danh của host; chạy trong Tomcat | Satellite |
| Pulp 3 | Đồng bộ, lưu và phân phối content (RPM, errata, container, file, Ansible collection); gói trùng giữa các repo chỉ lưu một lần; metadata nằm trong PostgreSQL | Satellite, Capsule |
| Smart Proxy (`foreman-proxy`) | API của Capsule (cổng 9090): DHCP, DNS, TFTP, template, đăng ký, OpenSCAP, REX, Ansible | Satellite (tích hợp), Capsule |
| Dynflow + Sidekiq | Điều phối tác vụ nền: sync, publish/promote, đồng bộ Capsule, REX | Satellite (trên Capsule là feature Dynflow chạy trong `foreman-proxy`) |
| PostgreSQL | CSDL `foreman`, `candlepin`, `pulpcore`; Capsule có CSDL `pulpcore` riêng | Satellite, Capsule |
| Redis | Bộ nhớ đệm cho Foreman và Pulp; Dynflow dùng để theo dõi tác vụ | Satellite, Capsule |
| Apache httpd | Kết thúc TLS 443/80, reverse proxy tới Foreman (Puma), Candlepin (`/rhsm`), Pulp (`/pulp/...`) | Satellite, Capsule |
| Puma | App server Ruby chạy Foreman (service `foreman`) | Satellite |
| OpenVox server (tuỳ chọn) | Máy chủ Puppet/OpenVox, cổng 8140; tắt mặc định | Satellite, Capsule |
| Mosquitto (tuỳ chọn) | MQTT broker cho REX pull mode, cổng 1883 | Capsule hoặc Satellite bật pull REX |

## Dịch vụ hệ thống và vai trò

| Service (systemd) | Có trên | Vai trò | Dấu hiệu thường gặp khi lỗi |
|---|---|---|---|
| `httpd` | Satellite, Capsule | Cổng vào 443/80 cho Web UI, API, RHSM, content | Web UI/`dnf`/đăng ký báo 503 hoặc không kết nối |
| `foreman` | Satellite | Puma chạy Web UI và API | Web UI 503, `hammer` lỗi |
| `dynflow-sidekiq@orchestrator`, `dynflow-sidekiq@worker-N`, `dynflow-sidekiq@worker-hosts-queue-N` | Satellite | Lập lịch và thực thi tác vụ nền | Tác vụ đứng ở trạng thái planned/pending |
| `tomcat` | Satellite | Chạy Candlepin | Đăng ký host, import manifest lỗi; `hammer ping` báo candlepin FAIL |
| `postgresql` | Satellite, Capsule | CSDL | Gần như mọi chức năng lỗi |
| `redis` | Satellite, Capsule | Cache, trạng thái tác vụ | Tác vụ, Web UI chậm/lỗi |
| `pulpcore-api` | Satellite, Capsule | API của Pulp | Sync, publish, đồng bộ Capsule lỗi |
| `pulpcore-content` | Satellite, Capsule | Trả gói cho host và Capsule | `dnf` trên host báo 404/5xx |
| `pulpcore-worker@N` | Satellite, Capsule | Worker xử lý tác vụ Pulp | Tác vụ Pulp chờ mãi |
| `foreman-proxy` | Satellite, Capsule | Smart Proxy API 9090, template 8000, register endpoint | Refresh Capsule lỗi, đăng ký qua Capsule lỗi |
| `mosquitto` | Khi bật REX pull | MQTT 1883 | Job REX pull không tới host |

Quản lý dịch vụ (chạy bằng root trên Satellite hoặc Capsule):

```bash
satellite-maintain service list
satellite-maintain service status -b          # bản tóm tắt
satellite-maintain service status --only postgresql
satellite-maintain service restart
satellite-maintain health check
hammer ping                                   # chỉ trên Satellite: database, candlepin, pulp3, foreman_tasks
```

- Dùng `satellite-maintain service stop|start|restart` thay vì `systemctl` từng unit để dịch vụ dừng/khởi động đúng thứ tự (cơ bản về systemd: [systemd cơ bản](https://github.com/lovelybugf/kb-base-devops/blob/main/linux/systemd-basics.md)).
- Log chính: `/var/log/foreman/production.log`, `/var/log/foreman-proxy/proxy.log`, `/var/log/candlepin/`, `journalctl -u 'pulpcore*'`, `/var/log/httpd/`, `/var/log/foreman-installer/satellite.log`.

## Capsule: tính năng và kiểu Capsule

Nhóm content:
- **Repository synchronization**: kéo content của các Lifecycle Environment được gán từ Satellite.
- **Content delivery**: host gán cho Capsule tải gói từ Capsule.
- **Host action delivery**: thực thi job đã lập lịch (REX) trên host.
- **RHSM proxy**: host đăng ký và làm việc subscription qua Capsule, không cần tới Satellite hay Customer Portal.

Nhóm hạ tầng: DHCP (ISC, Active Directory, libvirt, Infoblox), DNS (BIND, Active Directory), TFTP, Realm (IdM/AD), OpenVox server và Puppet CA, BMC (IPMI/Redfish), proxy template provisioning, OpenSCAP, Remote Execution.

| Kiểu Capsule | Feature bật | Dùng khi |
|---|---|---|
| Infrastructure | DNS + DHCP + TFTP | Cấp phát host trong một mạng |
| Content | Pulp | Chỉ phân phối content gần host |
| Configuration | Pulp + Puppet + Puppet CA | Content và quản lý cấu hình |
| Đầy đủ | DNS + DHCP + TFTP + Pulp + Puppet + Puppet CA | Cô lập hoàn toàn một vùng mạng, host chỉ cần tới một điểm |

```bash
hammer capsule list
hammer capsule info --name "<CAPSULE_FQDN>"            # xem Features, LE đã gán
hammer capsule refresh-features --name "<CAPSULE_FQDN>"
```

Trong triển khai thực tế, Capsule chỉ dùng cho content và đăng ký thường hiển thị các feature Pulpcore, Registration, Templates, Openscap, Logs, Container_Gateway, Dynflow, Script (Script là plugin REX, có mặt kể cả khi không dùng REX).

## Mô hình connected và disconnected

| Mô hình | Satellite lấy content từ | Ghi chú |
|---|---|---|
| Connected | Red Hat CDN, trực tiếp hoặc qua HTTP proxy (443) | Nhận errata ngay khi Red Hat phát hành; chỉ Satellite cần ra Internet |
| Disconnected — ISS Network Sync | Satellite upstream (connected) qua HTTPS nội bộ | Downstream không ra Internet; chạy được giữa các phiên bản có hỗ trợ |
| Disconnected — ISS Export Sync (air-gapped) | Gói export mang vật lý sang rồi import | Mọi Satellite phải cùng phiên bản |
| Custom CDN | Web server nội bộ chứa cây repo Red Hat | Satellite tự nhận ra repo nào trong manifest có trên CDN nội bộ |

- Satellite disconnected nên tắt **Subscription Connection** để không gọi Red Hat Portal; đổi lại không refresh manifest trực tiếp được.
- ISS chỉ dùng giữa các Satellite; Capsule đồng bộ từ Satellite bằng cơ chế riêng.
- Host sau Satellite disconnected vẫn nhận errata mới nhất, miễn là content đã được nhập vào.
- **Lưu ý từ dự án:** "cài đặt disconnected" (cài gói Satellite từ Binary DVD/ISO, OS Satellite không đăng ký RHSM) khác với "content disconnected". Có triển khai cài offline nhưng Satellite vẫn sync từ CDN qua một đường Internet mở riêng cho nó — về content đó là mô hình connected. Ghi rõ cả hai khía cạnh trong tài liệu thiết kế để tránh hiểu nhầm.

## Giới hạn hỗ trợ và bảo mật cần biết trước khi cài

- Máy Satellite/Capsule chỉ chạy Satellite/Capsule; bản RHEL đi kèm subscription Satellite chỉ được dùng cho mục đích này.
- Không thao tác trực tiếp vào CSDL hay API nội bộ của Pulp, Candlepin, Tomcat — chỉ qua Web UI, Hammer, API của Satellite; làm sai có thể hỏng CSDL không phục hồi.
- Chỉ dùng plugin Foreman đóng gói kèm Satellite. Tính năng cho hệ điều hành khác RHEL có sẵn nhưng chỉ được hỗ trợ với RHEL.
- Client được quản lý content: RHEL 10 và 9 (x86_64, ppc64le, s390x, aarch64), RHEL 8 (x86_64, ppc64le, s390x), RHEL 7 (x86_64, ppc64, ppc64le, aarch64, s390x). Provisioning chỉ x86_64. OpenVox agent chưa có cho RHEL 10.
- FIPS: phải bật trên RHEL trước khi cài, không bật được sau; Satellite chạy được trên RHEL FIPS nhưng bản thân không có chứng nhận FIPS. Crypto policy hỗ trợ `DEFAULT` và `FIPS`, không hỗ trợ `FUTURE`.
- SELinux: khuyến nghị enforcing; permissive vẫn cài được; disabled không được hỗ trợ. `fapolicyd` bật/tắt được bất cứ lúc nào.

## Tài khoản hệ thống do Satellite tạo

| User | UID/GID | Đặt trước được UID/GID |
|---|---|---|
| `foreman`, `foreman-proxy`, `unbound` | tự cấp | Có |
| `apache` | 48 | Không |
| `postgres` | 26 | Không |
| `puppet` | 52 | Không |
| `tomcat` | 53 | Không |
| `pulp` | tự cấp | Không |
| `saslauth` | GID 76 | Không |

Nếu OS dùng nguồn định danh ngoài (SSSD, AD, IdM), nguồn đó không được chứa tài khoản trùng tên `apache`, `foreman`, `foreman-proxy`, `postgres`, `pulp`, `puppet`, `redis`, `tomcat`; không đổi home/shell của các tài khoản này.

## Lưu ý thường gặp

- `hammer ping` chỉ phản ánh Satellite. Host không tải được gói thì kiểm tra `httpd` và `pulpcore-content` trên chính Capsule mà host dùng.
- Tài liệu khoá học cũ (bản 6.15 trở về trước) còn nhắc qpid (cổng 5646/5671) và MongoDB — Satellite 6.19 không còn các thành phần này.
- Cài thêm phần mềm, agent hay hardening lên máy Satellite trước khi cài là nguyên nhân lỗi phổ biến; xem [Lập kế hoạch triển khai](deployment-planning.md).

## Liên quan

- [Cổng mạng và chiều kết nối](network-ports.md)
- [Lập kế hoạch triển khai](deployment-planning.md)
- [Sizing và tuning](sizing-and-tuning.md)
- [Cài Satellite connected](../installation/install-satellite-connected.md) · [Cài Satellite disconnected](../installation/install-satellite-disconnected.md) · [Cài Capsule](../installation/install-capsule.md)
- [Inter-Satellite Sync](../content/inter-satellite-sync.md) · [Download policy](../content/download-policy.md)
- [Health check và xử lý sự cố](../operations/health-check-and-troubleshooting.md)

## Nguồn

- Red Hat Satellite 6.19, Overview, concepts, and deployment considerations, chương 1 Introduction (PDF tr. 8), chương 5 Major Satellite components (PDF tr. 19–21), chương 7 Supported usage and versions (PDF tr. 26–28), chương 8 Security considerations (PDF tr. 29–30), mục 11.1 Installing a Satellite Server (PDF tr. 42–43), phụ lục A Technical users (PDF tr. 65–66) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Administering Red Hat Satellite, phụ lục A Starting and stopping Satellite services (PDF tr. 118) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Installing Satellite Server in a disconnected network environment, mục 3.7, 5.2–5.4 (PDF tr. 22, 30–33); Installing Capsule Server, mục 1.1–1.2 (PDF tr. 7–8) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Tuning performance of Red Hat Satellite, mục 5.4, 5.7 (Dynflow, Redis; PDF tr. 16–17, 20) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- RH403 (Red Hat Satellite Administration), chương 1: danh sách service từ `satellite-maintain service list`, `hammer ping`.
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: danh sách feature của Capsule chỉ dùng cho content; phân biệt cài offline và content disconnected.
