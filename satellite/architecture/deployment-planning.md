---
title: Lập kế hoạch triển khai Satellite
tags: [concept, satellite, installation, capsule, content-view]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Overview, concepts, and deployment considerations, chương 11–12 (PDF tr. 42–61); Configuring Capsules with a load balancer, chương 1 (PDF tr. 8–9); RH403 chương 1; tài liệu thiết kế dự án (ẩn danh)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Trước khi cài, chốt mô hình content, số Capsule theo vị trí và vùng mạng, cấu trúc Organization/Location, đường Lifecycle Environment và các quyết định không đổi được sau cài như FIPS. Phần khuyến nghị của Red Hat lấy từ Overview chương 11–12; mục "Bài học từ thiết kế dự án" là kinh nghiệm đã áp dụng, ẩn danh.

## Câu hỏi phải trả lời trước khi cài

| Câu hỏi | Ảnh hưởng | Ghi chú |
|---|---|---|
| Satellite có được ra Red Hat CDN không (trực tiếp, qua proxy, hay không)? | Connected, ISS Network Sync hay ISS Export Sync | Xem [Thành phần và mô hình kết nối](components-and-services.md) |
| Bao nhiêu host, ở những vị trí/vùng mạng nào? | Số Capsule, tuning profile, sizing | Ít nhất một Capsule mỗi vị trí địa lý |
| Cần HA cho đăng ký và content không? | Capsule sau load balancer | Chỉ áp dụng cho Capsule cài mới |
| Một hay nhiều Organization/manifest? | Cấu trúc Org, subscription | Mỗi Organization cần một manifest |
| CSDL nội bộ hay PostgreSQL ngoài? | Tải, vận hành CSDL | Red Hat không hỗ trợ bảo trì CSDL ngoài |
| Bật tính năng nào: provisioning, REX, OpenSCAP, Lightspeed, Puppet? | Cổng mạng, feature của Capsule | Tính năng ngoài phạm vi thì không mở cổng |
| FIPS, crypto policy, IPv4 hay IPv6? | Không đổi được sau cài | FIPS chỉ bật trước khi cài; không hỗ trợ dual-stack |
| Chứng chỉ mặc định hay CA nội bộ? | Phân phối CA tới host | Quyết định từ đầu để khỏi triển khai lại CA trên host |

## Lộ trình triển khai theo Red Hat

1. Cài Satellite trên máy RHEL 9 mới, chuyên dụng — connected hoặc disconnected; tuỳ chọn CSDL ngoài ([cài connected](../installation/install-satellite-connected.md), [cài disconnected](../installation/install-satellite-disconnected.md)).
2. Cấu hình xác thực ngoài nếu cần: AD (username/password, SSO), IdM, Red Hat SSO (thêm OTP/TOTP), LDAP ([xác thực ngoài](../security/external-authentication.md)).
3. Lập kế hoạch Organization và Location (mục bên dưới).
4. Cài Capsule cho từng vị trí/vùng mạng, quyết định Capsule nào chạy DNS/DHCP/TFTP hay dùng dịch vụ ngoài ([cài Capsule](../installation/install-capsule.md)).
5. Nhập manifest cho từng Organization ([manifest](../content/subscriptions-manifest.md)).
6. Xây Library: bật repo Red Hat, tạo product/repo tuỳ chỉnh, sync và sync plan ([repository và sync](../content/repositories-and-sync.md)).
7. Định chiến lược truy cập content cho host bằng Content View và Lifecycle Environment ([CV và LE](../content/content-views-and-lifecycle.md)).
8. Thiết kế RBAC: role theo hạ tầng, theo giai đoạn vòng đời, theo nhiệm vụ ([user, role, org](../security/users-roles-orgs.md)).
9. Cấu hình provisioning nếu dùng ([tổng quan provisioning](../provisioning/provisioning-overview.md)).
10. Chọn phương án DR và sao lưu ([backup và restore](../operations/backup-and-restore.md)).
11. Cấu hình theo dõi subscription: system purpose; với connected thêm virt-who, inventory upload ([virt-who](../content/virt-who.md)).
12. Tính năng bổ sung: REX (pull qua MQTT/HTTPS hoặc push qua SSH), Ansible/Puppet, OpenSCAP, load balancer cho Capsule, Lightspeed.

## Các kịch bản triển khai phổ biến

| Kịch bản | Khi nào dùng | Điểm thiết kế |
|---|---|---|
| Một Satellite, Capsule tích hợp | Một vị trí, ít host | Đơn giản nhất; host tới thẳng Satellite |
| Một vị trí, nhiều subnet cô lập | Nhiều mạng tách biệt trong cùng DC | Khuyến nghị một Capsule quản lý các subnet tách biệt thay vì nhiều Capsule DHCP/DNS |
| Nhiều vị trí | Chi nhánh, nhiều DC | Ít nhất một Capsule mỗi vị trí để host lấy content cục bộ, tiết kiệm WAN; Capsule chỉ nên phục vụ host ở gần |
| Capsule theo vùng mạng | DMZ, vùng đối tác, vùng production/dev tách biệt | Mỗi vùng một Capsule; host của vùng chỉ cần tới Capsule của vùng |
| Capsule sau load balancer | Cần HA và phân tải cho đăng ký, content | Xem mục dưới |
| Disconnected — ISS Network Sync | Downstream không ra Internet nhưng tới được upstream | Upstream connected sync CDN, downstream sync qua HTTPS |
| Disconnected — ISS Export Sync | Air-gapped hoàn toàn | Export → mang vật lý → import; mọi Satellite cùng phiên bản |
| Nhiều manifest | Nhiều tài khoản Red Hat hoặc nhiều đơn vị | Mỗi manifest một Organization |
| Satellite trên AWS | Host trên cloud hoặc hybrid | VPN site-to-site giữa on-prem và region; không có VPN thì dùng tên DNS ngoài khi đăng ký Capsule; dữ liệu ra khỏi region bị tính phí |

Số Capsule mỗi Satellite không có giới hạn cứng; tài liệu hãng ghi nhận đã thử một Satellite phục vụ 17 Capsule (cấu hình 2 vCPU).

### Capsule sau load balancer

- Cân bằng tải được cho: đăng ký host, cung cấp content, cấu hình bằng Puppet. Provisioning, virt-who, REX vẫn đi thẳng tới từng Capsule.
- Mọi Capsule trong pool phải có cùng content (cùng LE, cùng phiên bản CV đã sync); nâng cấp lần lượt từng Capsule.
- Load balancer TCP (L4), SSL pass-through, giữ phiên theo IP nguồn cho 443; chứng chỉ Capsule có SAN chứa FQDN của VIP. Chi tiết: [Capsule sau load balancer](../installation/capsule-load-balancer.md).

## Quy hoạch Organization và Location

- **Organization**: đơn vị nghiệp vụ, phòng ban, nhóm — vùng cô lập cấu hình và content. Mỗi Organization cần manifest riêng. Label của Organization đầu tiên lấy theo `--foreman-initial-organization` và không đổi được (tên thì đổi được) — đặt tên có nghĩa ngay khi cài.
- **Location**: nơi đặt vật lý (quốc gia, thành phố, DC), có thể lồng nhau. Subnet, domain, compute resource không gán trực tiếp cho Capsule mà gán cho Location.
- Người dùng và tài nguyên chỉ thấy trong context của mình, nên Org/Location cũng là một phần của phân quyền.
- Không trộn tài nguyên khác context trong một Host Group (ví dụ subnet của Location này với compute resource của Location khác) — Host Group sẽ không hợp lệ.
- Thực tế với một đơn vị sở hữu: thường **một Organization**, một Location theo DC; tách DEV/PROD bằng Lifecycle Environment và Activation Key thay vì tạo thêm Organization.

## Chiến lược content ở mức thiết kế

Đường Lifecycle Environment:
- Không gán host trực tiếp vào **Library** (Library nhận content mới nhất, không có bước kiểm thử). Không gán Library cho Capsule (Capsule sẽ sync liên tục mỗi lần repo đổi).
- Một LE duy nhất (Library → PROD): đơn giản, kiểm thử trong Library.
- Một đường chung nhiều bước (ví dụ Library → DEV → QA → PROD): kiểm thử kỹ, tốn công promote.
- Mỗi ứng dụng một đường riêng: chu kỳ phát hành độc lập, bảo trì phức tạp.

Content View:

| Cách chia | Ưu | Nhược |
|---|---|---|
| Một CV cho tất cả | Ít CV, tiết kiệm thời gian và dung lượng | Mọi thay đổi ảnh hưởng nhiều host, khó lọc |
| Mỗi loại host một CV | Hợp khi ít loại host (tới khoảng 30) | Không chia sẻ được content chung; vá khẩn phải cập nhật mọi CV |
| Mỗi loại host một Composite CV | Tách OS và ứng dụng, cập nhật với tần suất khác nhau | Nhiều đối tượng cần quản lý |
| CV theo thành phần (ví dụ CSDL) ghép vào nhiều CCV | Chuẩn hoá cao | Số CV tăng |

- Tránh tạo quá nhiều CV; CV càng lớn, publish/promote càng lâu. Lọc gói thì phải giữ đủ dependency.
- Không đưa repo kickstart vào CV — chỉ dùng cho provisioning.
- Mẫu đã dùng thực tế: mỗi bản RHEL major một CV (BaseOS + AppStream + Satellite Client 6); một CV riêng cho OS của Capsule (RHEL 9 + Capsule + Maintenance + Client); đường Library → DEV → PROD (hoặc thêm UAT/STAG); errata filter theo ngày cho mốc vá hằng tháng; vá khẩn bằng incremental update; giữ khoảng 3 phiên bản gần nhất mỗi CV.

Host Group: cấu trúc phẳng (ít loại host), theo LE (mỗi giai đoạn có chủ sở hữu riêng), theo ứng dụng, hoặc theo Location/Capsule; Host Group lồng nhau kế thừa tham số ([host group và tham số](../hosts/host-groups-and-parameters.md)).

## Phương án DR

| Phương án | Cách làm | Mất dữ liệu khi khôi phục |
|---|---|---|
| Snapshot VM | Snapshot Satellite bằng công cụ hypervisor | Từ thời điểm snapshot |
| Active/passive, lưu trữ ngoài | `/var/lib/pulp` và `/var/lib/pgsql` trên NAS nhân bản sang DC khác; Satellite passive là bản clone; chuyển DNS khi sự cố | Theo chu kỳ nhân bản |
| Active/passive, backup/restore | Sao lưu định kỳ, restore sẵn lên Satellite passive; chuyển DNS | Theo tần suất backup và thời gian restore |
| Hai Satellite active | Mỗi DC một Satellite; tự động đăng ký lại host sang Satellite kia khi sự cố | Phải đồng bộ CV giữa hai bên để tránh lệch content |

## Bài học từ thiết kế dự án (ẩn danh)

Quyết định thiết kế đã dùng và lý do:
- **Satellite ở DMZ (hoặc vùng có đường ra Internet), Capsule ở vùng nội bộ**: chỉ Satellite cần ra CDN; host nghiệp vụ chỉ tới Capsule, không cần Internet. Đường ra Internet của Satellite có thể là forward proxy whitelist hoặc firewall allow-list trực tiếp.
- **Hai Capsule active–active sau load balancer, định cỡ N+1**: mỗi Capsule một mình phục vụ được toàn bộ host; hai VM đặt khác host vật lý/datastore (anti-affinity).
- **Host chỉ dùng FQDN của VIP**: registration URL và template URL của Capsule trỏ về VIP, nên thay/thêm Capsule không phải cấu hình lại host.
- **Capsule riêng cho DMZ và vùng đối tác**, không qua load balancer, chỉ phục vụ LE PROD; chiều Capsule DMZ → Satellite cần phê duyệt ngoại lệ của bộ phận bảo mật.
- **Simple Content Access** cho mọi Organization; không bao giờ xoá manifest, đổi subscription thì refresh.
- **Download policy** là đánh đổi: Immediate khi đường ra CDN bị giới hạn khung giờ (gói có sẵn cục bộ), On Demand khi muốn tiết kiệm dung lượng ([download policy](../content/download-policy.md)).
- **Tính năng ngoài phạm vi được ghi rõ** (provisioning, REX, Lightspeed, Puppet) kèm lý do và cổng phải mở nếu bật sau; không mở cổng cho tính năng không dùng.
- **Quyết định không đổi được ghi thành mục riêng**: FIPS, IPv4 (giữ IPv6 loopback), label Organization, phương án chứng chỉ.

Chuẩn bị môi trường (bàn giao checklist cho chủ hạ tầng trước ngày cài):
- VM RHEL 9 bản minor mới nhất, cài `@Base`, **chưa hardening, chưa cài antivirus/EDR/agent giám sát**. Nếu chính sách bắt buộc: cài và kiểm tra Satellite trước → sao lưu → áp hardening/agent với danh sách loại trừ thư mục, tiến trình, cổng → kiểm tra lại.
- Bản ghi DNS A và PTR cho Satellite, từng Capsule và VIP; split DNS nếu DMZ dùng DNS riêng; NTP; root/sudo, umask 0022; SELinux không disabled; `/tmp` mount `exec`.
- Firewall theo ma trận ([cổng mạng](network-ports.md)), proxy không SSL inspection, LB đủ listener.
- Bộ lệnh tự kiểm tra trước ngày cài: `hostname -f`, `dig +short $(hostname -f)`, `dig +short -x <IP>`, `chronyc sources`, `getenforce`, `umask`, `fips-mode-setup --check`, `update-crypto-policies --show`, `findmnt /tmp`, kiểm tra cổng giữa các máy.

Triển khai theo giai đoạn: chuẩn bị → cài Satellite → cấu hình content → cài Capsule → cấu hình LB → tích hợp (người dùng, SMTP, sao lưu) → thí điểm (tối đa khoảng 5 host mỗi môi trường) và UAT → bàn giao. Các ca UAT nên có: sync qua proxy, publish/promote và Capsule tự sync đúng LE, đăng ký qua VIP, cài gói không cần Internet, failover/failback Capsule, báo cáo errata, OpenSCAP, đăng nhập theo nhóm quyền, sao lưu, host bị chặn khi đi thẳng tới Capsule/Satellite.

## Liên quan

- [Thành phần và dịch vụ](components-and-services.md) · [Cổng mạng](network-ports.md) · [Sizing và tuning](sizing-and-tuning.md)
- [Capsule sau load balancer](../installation/capsule-load-balancer.md) · [Inter-Satellite Sync](../content/inter-satellite-sync.md)
- [CV và Lifecycle Environment](../content/content-views-and-lifecycle.md) · [Errata và incremental update](../content/errata-and-incremental-update.md)
- [Disaster recovery và clone](../operations/disaster-recovery-and-cloning.md)

## Nguồn

- Red Hat Satellite 6.19, Overview, concepts, and deployment considerations, chương 11 Deployment path for Red Hat Satellite (PDF tr. 42–51) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Overview, concepts, and deployment considerations, chương 12 Common deployment scenarios (PDF tr. 52–61) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Configuring Capsules with a load balancer, chương 1 Overview of load balancing (PDF tr. 8–9) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Installing Satellite Server in a connected network environment, mục 1.1–1.2, 3.6 (PDF tr. 8–9, 20) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- RH403 (Red Hat Satellite Administration), chương 1: câu hỏi trước triển khai, kịch bản Capsule cục bộ/từ xa.
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: quyết định thiết kế Satellite DMZ + Capsule sau LB, Capsule theo vùng mạng, checklist chuẩn bị, giai đoạn triển khai và danh mục ca UAT đã áp dụng cho các dự án đã nghiệm thu.
