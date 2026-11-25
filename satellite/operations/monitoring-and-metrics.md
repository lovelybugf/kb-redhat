---
title: Giám sát Satellite, Capsule và usage metrics
tags: [howto, satellite, monitoring, capsule, performance]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Administering Red Hat Satellite, chương 4.5, 5, 18, Phụ lục G (PDF tr. 30–35, 99–104, 158–163); kinh nghiệm triển khai thực tế (ẩn danh)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Giám sát trên Web UI qua Monitor > Dashboard, Satellite Tasks, Audits và trang Capsules; phía máy chủ dựa vào `satellite-maintain health check`, `hammer ping` và dung lượng đĩa. Usage metrics gửi Red Hat qua hai systemd timer, xem bằng `satellite-maintain report generate`, tắt bằng mask timer.

## Giám sát trên Web UI

### Monitor > Dashboard

Kéo thả để sắp xếp widget. Widget nên xem hằng ngày:

| Widget | Nhìn gì |
|---|---|
| Task Status, Latest Warning/Error Tasks | Task lỗi, task cảnh báo mới nhất |
| Sync Overview | Product/repo chưa sync hoặc sync lỗi |
| Host Configuration Status / Chart | Host lỗi, Out of sync, No reports, Alerts disabled |
| Latest Errata | Errata áp dụng được cho host |
| Content Views | Trạng thái publish của CV |
| Virt-who Configuration Status | No Reports (cấu hình chưa triển khai hoặc virt-who không tới được Satellite), No Change, OK |
| Latest Compliance Reports, Compliance Reports Breakdown | Kết quả OpenSCAP (pass/fail/other) |
| Red Hat Lightspeed Actions, Risk Summary | Khuyến nghị theo mức rủi ro (nếu dùng) |
| Latest Events, New Hosts, Discovered Hosts | Sự kiện bất thường, host mới |

### Monitor > Satellite Tasks > Tasks

Theo dõi sync, publish, áp errata; lọc theo trạng thái, xem thời gian chạy, Cancel/Resume. Task chạy trên Dynflow; timeout của task tới Capsule/host chỉnh bằng setting `%_timeout` (xem [Health check và xử lý sự cố](health-check-and-troubleshooting.md)).

### Monitor > Audits

Ghi lại thay đổi của mọi user, dùng cho truy vết và điều tra. Danh sách thuộc tính audit: `foreman-rake audits:list_attributes`. Dọn audit cũ: [Quản lý dung lượng](storage-management.md).

### Infrastructure > Capsules

- Cột Status: xanh là chạy, đỏ là Capsule không hoạt động (khởi động lại dịch vụ `foreman-proxy` trên Capsule).
- Cột Features: dịch vụ Capsule đang cung cấp (Pulp, DHCP, DNS, TFTP, REX, OpenSCAP...). Sau khi đổi cấu hình Capsule chọn **Refresh** trong cột Actions.
- Chọn Any Organization để thấy mọi Capsule.
- Vào từng Capsule: tab Overview (số host quản lý, dung lượng Pulp đã dùng và còn trống), tab Services (domain DNS, số Pulp worker...), tab Puppet/Puppet CA nếu bật Puppet.

## Giám sát phía máy chủ

| Chỉ số | Cách lấy | Ngưỡng / kỳ vọng | Tần suất |
|---|---|---|---|
| Dịch vụ | `satellite-maintain service status -b` | All services running | Hằng ngày |
| Thành phần | `hammer ping` | Mọi mục ok | Hằng ngày |
| Health check | `satellite-maintain health check` | Không FAIL | Hằng ngày |
| Task paused/error | `hammer task list --search 'state = paused or result = error'` | Không có | Hằng ngày |
| Sync repo | Content > Sync Status, widget Sync Overview | Không lỗi, đúng lịch sync plan | Hằng ngày |
| Capsule | `hammer capsule list`, trang Capsules | Tất cả Active, content đã sync | Hằng ngày |
| Dung lượng | `df -h /var/lib/pulp /var/lib/pgsql /var/log <BACKUP_MNT>` | Cảnh báo 80 %; nơi lưu backup ≤ 70 % | Hằng tuần |
| Sao lưu | Log script backup | Mỗi ngày một dòng hoàn tất | Hằng tuần |
| Host check-in | All Hosts, cột Last checkin / Global status | Host không check-in lâu cần xem lại | Hằng tuần |
| Đồng bộ thời gian | `chronyc tracking` | Lệch nhỏ (lệch lớn gây lỗi SSL) | Hằng tuần |
| Hạn chứng chỉ | `openssl x509 -noout -enddate -in <CERT_FILE>` | Còn hạn, gia hạn trước | Hằng quý |
| Diễn tập restore | Restore bản full lên VM cách ly | `hammer ping` ok, dữ liệu đủ | 6–12 tháng, trước nâng cấp lớn |

Bảng trên là danh mục kiểm tra định kỳ đã dùng trong triển khai thực tế, các lệnh lấy từ tài liệu hãng. Global status "Error" của host thường chỉ do có errata bảo mật áp dụng được, không có nghĩa host mất kết nối.

### Tự động hoá kiểm tra

Hãng khuyên tự động chạy `satellite-maintain health check` (đặc biệt khi có hai Satellite cùng active). Ví dụ cron gửi thư khi có lỗi, **chưa chạy thử**: giả định đầu ra có chữ `FAIL` khi có mục lỗi; kiểm lại trên máy trước khi dùng. Cần gửi thư được (xem [Email và webhook](notifications-and-webhooks.md)).

```bash
cat > /usr/local/sbin/sat-health.sh << 'EOF'
#!/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
OUT=$(satellite-maintain health check 2>&1; hammer ping 2>&1)
if echo "$OUT" | grep -q 'FAIL'; then
  echo "$OUT" | mail -s "[Satellite] health check loi tren $(hostname -f)" <RECIPIENT>
fi
EOF
chmod 700 /usr/local/sbin/sat-health.sh
echo '*/30 * * * * root /usr/local/sbin/sat-health.sh' > /etc/cron.d/sat-health
```

`hammer` chạy bằng cron cần thông tin đăng nhập trong cấu hình hammer của root; giờ chạy nên tránh khung sao lưu offline (lúc đó dịch vụ dừng có chủ đích). Cảnh báo theo sự kiện (host đổi trạng thái, job REX xong) thì dùng webhook sự kiện **Status Changed** đẩy sang hệ thống giám sát. Cảnh báo theo ngưỡng CVE dùng report template lên lịch: [Report template và cảnh báo CVE](../hosts/report-templates-and-cve-alert.md).

## Usage metrics

Satellite thu dữ liệu sử dụng (tính năng dùng, số đối tượng trong CSDL) để Red Hat ưu tiên phát triển và hỗ trợ debug:

| Timer | Làm gì | Kết quả |
|---|---|---|
| `satellite-usage-metrics-generate.timer` | Chạy `satellite-maintain report generate` hằng tuần | `/var/lib/foreman-maintain/satellite_metrics.yml` |
| `satellite-usage-metrics-condense.timer` | Rút gọn để tải lên như custom fact của RHSM, hằng tuần | `/etc/rhsm/facts/foreman.facts` |

Cả hai file được đưa vào `sos report`. Xem nội dung, chạy bằng root:

```bash
satellite-maintain report generate
satellite-maintain report condense
```

Tắt thu thập: mask timer và xoá file đã sinh.

```bash
systemctl list-timers 'satellite-usage-metrics*'
systemctl mask satellite-usage-metrics-generate.timer
rm /var/lib/foreman-maintain/satellite_metrics.yml
systemctl mask <CONDENSE_TIMER>
rm /etc/rhsm/facts/foreman.facts
systemctl status satellite-usage-metrics-generate.timer
```

Tài liệu 6.19 ghi tên timer rút gọn lúc là `satellite-usage-metrics-condense.timer`, lúc là `...-condensed.timer`; lấy tên đúng từ `systemctl list-timers` rồi thay vào `<CONDENSE_TIMER>`.

Nội dung thu (Phụ lục G): cách nhóm tài nguyên (org, location, host group lồng nhau), dùng OpenSCAP, recurring logic, virt-who và loại hypervisor, compute resource VMware, lab features, môi trường disconnected, RBAC và custom role, nguồn xác thực (LDAP, Kerberos, Keycloak/OIDC), ACS, số repo theo loại, số host theo HĐH/loại, Personal Access Token, SELinux enforcing, bookmark, webhook, mạng IPv4/IPv6, image mode, phiên bản. Không có mật khẩu hay nội dung gói.

## Liên quan

- [Health check và xử lý sự cố](health-check-and-troubleshooting.md)
- [Quản lý dung lượng lưu trữ](storage-management.md)
- [Email thông báo và webhook](notifications-and-webhooks.md)
- [Sizing và tuning](../architecture/sizing-and-tuning.md)
- [virt-who](../content/virt-who.md)
- [OpenSCAP compliance](../security/openscap-compliance.md)

## Nguồn

- Red Hat Satellite 6.19, Administering Red Hat Satellite, chương 18 Monitoring Satellite resources (PDF tr. 99–104), chương 5 Usage metrics collection (PDF tr. 33–35), mục 4.5 Disaster recovery with two active Satellite Servers (PDF tr. 30–32), Phụ lục G Usage metrics collected by Satellite (PDF tr. 158–163) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: danh mục kiểm tra định kỳ (dịch vụ, task, dung lượng, log sao lưu, chứng chỉ, diễn tập restore); Global status Error do errata, không phải mất kết nối.
