---
title: Cổng mạng và chiều kết nối của Satellite, Capsule và host
tags: [reference, satellite, networking, capsule, provisioning, remote-execution]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Overview, concepts, and deployment considerations, chương 9 và 13 (PDF tr. 31–40, 62–64); Installing Satellite Server in a connected network environment, mục 1.5, 2.1, 3.1 (PDF tr. 11, 14, 17); Installing Capsule Server, mục 1.5 (PDF tr. 9–10)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Host chỉ cần tới Capsule của mình qua 443, 80 và 9090 (thêm 8000, 1883, 8140, 67–69 khi dùng tính năng tương ứng); chỉ Satellite đi ra Red Hat CDN qua 443. Satellite và Capsule phải mở hai chiều: Capsule → Satellite 443/80 và Satellite → Capsule 443/9090.

## Cách đọc bảng

- Mỗi dòng ghi **chiều mở kết nối** `Nguồn → Đích`; gói trả về tự được cho phép trên firewall stateful.
- Host đăng ký qua Capsule thì mọi giao tiếp với Satellite đi qua Capsule: ở vùng mạng xa chỉ cần mở tới Capsule, host không cần thấy Satellite.
- Satellite có Capsule tích hợp, nên host đăng ký thẳng vào Satellite dùng đúng các cổng "host → Capsule" nhưng đích là Satellite. Bản thân OS của Capsule là host của Satellite.

```text
             Red Hat CDN / RHSM
                    ▲ 443 (trực tiếp hoặc qua HTTP proxy)
                    │
              Satellite Server ──── 443, 9090 ────►  Capsule
                    ▲                                 │  ▲
                    └────────── 443, 80 ──────────────┘  │ 443, 80, 9090 (+8000, 1883, 8140, 67–69)
                                                         │
                                 Capsule ── 22 (REX push) ──►  Host
```

## Satellite ↔ Internet

| Nguồn → Đích | Cổng | Mục đích | Mức |
|---|---|---|---|
| Satellite → `subscription.rhsm.redhat.com` | 443/TCP | RHSM, manifest, subscription | Bắt buộc (connected) |
| Satellite → `cdn.redhat.com` | 443/TCP | Sync repo Red Hat | Bắt buộc (connected) |
| Satellite → `cert.console.redhat.com`, `api.access.redhat.com`, `cert-api.access.redhat.com`, `console.redhat.com`, `connect.cloud.redhat.com` | 443/TCP | Red Hat Lightspeed | Tuỳ chọn |
| Satellite → `*.akamaiedge.net` | 443/TCP | Kéo container image để cài Lightspeed trong Satellite | Tuỳ chọn |
| Satellite → repo tuỳ chỉnh | 80 hoặc 443/TCP | Sync repo bên thứ ba | Theo nhu cầu |
| Satellite downstream → Satellite upstream | 443/TCP | ISS Network Sync (disconnected) | Khi dùng ISS |
| Capsule → Red Hat Portal | 443/TCP | Gửi sosreport hỗ trợ case | Tuỳ chọn |

- Proxy cho OS Satellite đặt trong `/etc/rhsm/rhsm.conf` (`proxy_hostname`, `proxy_port`); proxy cho sync đặt trong Satellite (HTTP Proxies, Default HTTP Proxy). Khai báo proxy bằng FQDN, không dùng địa chỉ IPv6 trực tiếp.
- CDN chỉ phục vụ qua IPv4: Satellite trong mạng IPv6 bắt buộc dùng HTTP proxy dual-stack. Satellite không hỗ trợ cài dual-stack.
- Firewall lọc theo IP thì dùng danh sách "Public CIDR Lists for Red Hat" trên Customer Portal và cập nhật định kỳ.
- **Kinh nghiệm:** kết nối tới CDN dùng chứng chỉ client lấy từ manifest, nên proxy/firewall không được SSL inspection các tên miền Red Hat. Kiểm tra trước ngày cài:

```bash
curl -x http://<PROXY_FQDN>:<PROXY_PORT> -I https://cdn.redhat.com
curl -x http://<PROXY_FQDN>:<PROXY_PORT> -v https://subscription.rhsm.redhat.com 2>&1 | grep -i issuer
```

Kết quả đúng: có phản hồi HTTP (403 ở gốc `cdn.redhat.com` là bình thường) và issuer là CA công khai của Red Hat, không phải CA của proxy.

## Satellite ↔ Capsule

| Nguồn → Đích | Cổng | Mục đích |
|---|---|---|
| Capsule → Satellite | 443/TCP | Gọi API Satellite; đồng bộ content; chuyển tiếp yêu cầu của host (đăng ký, RHSM, facts); gửi kết quả REX, OpenSCAP; lấy template |
| Capsule → Satellite | 80/TCP | Lấy content và file qua HTTP |
| Satellite → Capsule | 443/TCP | Khởi tạo đồng bộ content (content mirroring) |
| Satellite → Capsule | 9090/TCP | Capsule API (Smart Proxy): refresh feature, ra lệnh sync, quản lý DHCP/DNS/TFTP |

- Cả hai chiều đều bắt buộc. Khi Satellite ở DMZ và Capsule ở vùng nội bộ, chiều Satellite → Capsule đi từ vùng kém tin cậy vào vùng tin cậy hơn: cần trình bộ phận bảo mật duyệt ngoại lệ sớm (kinh nghiệm dự án).
- Kết nối theo FQDN, phân giải xuôi/ngược được ở cả hai phía; tránh NAT/proxy làm sai lệch tên hay chứng chỉ giữa Satellite và Capsule.

## Host → Capsule (hoặc Satellite khi đăng ký trực tiếp)

| Cổng | Giao thức | Dùng cho | Khi nào cần |
|---|---|---|---|
| 443 | TCP HTTPS | Content (`/pulp/content`), RHSM proxy (`/rhsm`), đăng ký: gửi facts, danh sách gói, traces | Luôn cần |
| 80 | TCP HTTP | Content qua HTTP; tải CA (`/pub/katello-server-ca.crt`) khi đăng ký; trên Satellite còn nhận báo hoàn tất `/unattended/built` | Luôn cần |
| 9090 | TCP HTTPS | Register endpoint của Capsule, gửi báo cáo OpenSCAP, Discovery, thông báo REX pull | Đăng ký qua Capsule, OpenSCAP, Discovery |
| 8000 | TCP HTTP | Provisioning template, iPXE, UEFI HTTP Boot | Provisioning, lấy template |
| 1883 | TCP MQTT | Nhận thông báo job REX pull mode | Khi dùng REX pull |
| 8140 | TCP HTTPS | OpenVox agent | Khi dùng Puppet/OpenVox |
| 8443 | TCP HTTPS | Đăng ký kiểu cũ (deprecated), chỉ cho host triển khai trước các lần nâng cấp | Hạn chế mở |
| 53 | TCP/UDP | DNS do Capsule quản lý | Provisioning |
| 67 / 69 | UDP | DHCP / TFTP | PXE provisioning |

Quản trị viên: trình duyệt → Satellite 443; console máy ảo của compute resource qua 5910–5930/TCP.

## Satellite/Capsule → host và hạ tầng

| Nguồn → Đích | Cổng | Mục đích |
|---|---|---|
| Capsule (hoặc Satellite) → host | 22/TCP | REX push qua SSH (Script, Ansible) |
| Capsule → host đã discover | 8443/TCP | Gửi lệnh khởi động lại cho Discovery |
| Capsule DHCP → client | ICMP, 7/TCP echo | Kiểm tra IP còn trống trước khi cấp |
| Capsule → client | 68/UDP | DHCP |
| Capsule → BMC | 623 | Bật/tắt/khởi động lại máy qua IPMI |
| Satellite, Capsule → DNS server | 53/TCP, UDP | Phân giải tên; kiểm tra trùng bản ghi khi orchestration |
| Satellite, Capsule → DHCP server | 7911/TCP (OMAPI) | ISC DHCP local/remote (`--foreman-proxy-dhcp-server`) |
| Satellite, Capsule → Infoblox | 443/TCP | Quản lý DHCP lease trên Infoblox |
| Satellite → LDAP/AD | 389 hoặc 636/TCP | Xác thực người dùng ngoài |
| Satellite → libvirt | 22, 16514/TCP | Compute resource libvirt |
| Satellite → EC2, Azure, GCE | 443/TCP | Compute resource cloud |
| Satellite → OpenStack | 5000/TCP | Compute resource OpenStack |
| Satellite → hypervisor | 5900–5930/TCP | Console noVNC |

- VMware vCenter: Satellite gọi API vCenter qua HTTPS 443 (bảng của hãng không liệt kê riêng; đây là cổng API mặc định của vCenter).
- Dịch vụ dùng chung hay phải mở thêm (kinh nghiệm dự án): NTP 123/UDP, SMTP relay 25 hoặc 587/TCP, NFS sao lưu 2049/TCP, syslog/SIEM 514 hoặc 6514/TCP.

## Provisioning: PXE và HTTP Boot

- PXE: client cần UDP 67/68 tới DHCP (qua DHCP relay nếu khác subnet), UDP 69 tới TFTP trên Capsule, TCP 80 để tải kernel, initrd và kickstart.
- HTTP Boot (UEFI): UDP 67/68, TCP 8000 (HTTP) hoặc 9090 (HTTPS) tới Capsule; TFTP không dùng để boot nhưng Satellite vẫn dùng TFTP Capsule API để đặt cấu hình boot loader.
- Subnet provisioning phải gán đủ DHCP, TFTP, Templates (và HTTP Boot) Capsule; Satellite và Capsule phân giải được tên host sẽ cấp phát.

## firewalld trên Satellite và Capsule

Chạy bằng root trên Satellite (Capsule làm tương tự):

```bash
firewall-cmd --add-port="8000/tcp" --add-port="9090/tcp"
firewall-cmd \
  --add-service=dns --add-service=dhcp --add-service=tftp \
  --add-service=http --add-service=https --add-service=puppetmaster
firewall-cmd --runtime-to-permanent
firewall-cmd --list-all
```

Triển khai chỉ dùng content, đăng ký và OpenSCAP thường chỉ mở `https`, `http`, `9090/tcp` trên Satellite và thêm `8000/tcp` trên Capsule; mở `1883/tcp` khi bật REX pull.

## Kiểm tra nhanh

```bash
# Từ host tới Capsule
for p in 443 80 9090 8000; do timeout 3 bash -c "</dev/tcp/<CAPSULE_FQDN>/$p" && echo "$p open" || echo "$p closed"; done
# Từ Satellite tới Capsule
nc -zv -w3 <CAPSULE_FQDN> 443; nc -zv -w3 <CAPSULE_FQDN> 9090
# Từ Capsule tới Satellite
nc -zv -w3 <SATELLITE_FQDN> 443; nc -zv -w3 <SATELLITE_FQDN> 80
# Trạng thái firewalld trên Satellite/Capsule
firewall-cmd --state
```

## Lỗi hay gặp

- **443 thông nhưng 80/8000/9090 bị từ chối qua VIP**: load balancer mới tạo virtual server cho 443. Cần listener cho cả 443, 80, 8000, 9090 (xem [Capsule sau load balancer](../installation/capsule-load-balancer.md)).
- **REX, provisioning, virt-who qua load balancer không chạy**: các dịch vụ này không được cân bằng tải; host và Satellite phải tới thẳng từng Capsule.
- **Lỗi chứng chỉ hoặc sync thất bại sau khi qua firewall ứng dụng**: tắt SSL inspection và nhận diện ứng dụng, cho phép theo cổng/giao thức.
- **DHCP Capsule không cấp IP**: firewall trả lời *reject* cho ICMP hoặc port 7 bị coi là "IP đang có người dùng". Cho phép, hoặc *drop* chứ không *reject*.
- **firewalld bị tắt trên Capsule sau khi cài**: phát hiện khi UAT; luôn kiểm tra `firewall-cmd --state` và `--list-all` trong bước nghiệm thu.
- **Host vẫn ra được Internet** dù thiết kế chỉ cho phép đi qua Capsule: kiểm tra chặn egress của host như một ca kiểm thử riêng.
- Tài liệu khoá học cũ còn liệt kê 5646, 5671 (qpid): không dùng trên 6.19. Cổng 8443 cho đăng ký đã deprecated.
- REX pull: Mosquitto trên mỗi Capsule mặc định nhận tối đa 1024 host; tăng giới hạn xem [Sizing và tuning](sizing-and-tuning.md).

## Liên quan

- [Thành phần và dịch vụ](components-and-services.md)
- [Lập kế hoạch triển khai](deployment-planning.md)
- [Capsule sau load balancer](../installation/capsule-load-balancer.md) · [Cài Capsule](../installation/install-capsule.md)
- [Remote execution](../hosts/remote-execution.md) · [DNS, DHCP, TFTP và network boot](../provisioning/dns-dhcp-tftp-and-network-boot.md)
- [Đăng ký host](../hosts/host-registration.md)

## Nguồn

- Red Hat Satellite 6.19, Overview, concepts, and deployment considerations, chương 9 Networking considerations in Satellite: 9.1 Capsule networking, 9.2 Satellite Server port and firewall requirements, 9.3 Capsule port and firewall requirements (PDF tr. 31–40) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Overview, concepts, and deployment considerations, chương 13 Provisioning requirements: PXE booting, HTTP booting (PDF tr. 62–64) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Installing Satellite Server in a connected network environment, mục 1.5 IPv4 and IPv6 requirements (PDF tr. 11), 2.1 Opening required ports (PDF tr. 14), 3.1 Configuring the HTTP proxy to connect to Red Hat CDN (PDF tr. 17) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Installing Capsule Server, mục 1.5 Opening required ports (PDF tr. 9–10); Configuring Capsules with a load balancer, mục 1.2 (PDF tr. 8–9); Installing Satellite Server in a disconnected network environment, mục 5.4.1 (PDF tr. 31) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: kiểm tra proxy/SSL inspection bằng curl, LB thiếu listener 80/8000/9090, firewalld tắt trên Capsule, host còn ra Internet, các cổng dịch vụ dùng chung.
