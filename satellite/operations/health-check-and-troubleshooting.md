---
title: Health check, dịch vụ, log và xử lý sự cố Satellite
tags: [troubleshooting, satellite, monitoring, hammer, storage]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Administering Red Hat Satellite, Phụ lục A–C (PDF tr. 118–130), mục 4.2.3 và 18.2 (PDF tr. 25, 101–102); RH403 chương 11; kinh nghiệm triển khai và lab (ẩn danh)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Khi Satellite có vấn đề, chạy theo thứ tự `satellite-maintain service status -b`, `hammer ping`, `satellite-maintain health check`, rồi xem task paused/error và log. Restart bằng `satellite-maintain service restart`; Web UI trả 500 kèm SSH bị ngắt thường là sự cố storage, không phải lỗi ứng dụng.

## Bộ lệnh kiểm tra nhanh

Chạy bằng root trên Satellite (và trên từng Capsule, trừ `hammer`):

```bash
satellite-maintain service status -b
hammer ping
satellite-maintain health check
hammer task list --search 'state = paused or result = error' --per-page 20
df -hT; free -h; uptime
journalctl -p err -S "-1h" --no-pager | tail -50
chronyc tracking
```

| Lệnh | Kết quả tốt | Ghi chú |
|---|---|---|
| `satellite-maintain service status -b` | `All services are running [OK]` | `-b` in gọn; bỏ `-b` để xem `systemctl status` của từng dịch vụ |
| `hammer ping` | Mọi mục `Status: ok` | Gồm database, candlepin, candlepin_auth, candlepin_events, katello_events, pulp3, pulp3_content, foreman_tasks; trên lab 6.19 thấy 9 mục |
| `satellite-maintain health check` | Mọi mục `[OK]`, không FAIL | Kiểm tra số fact name trong CSDL, dịch vụ chạy, ping, task paused, Satellite có tự đăng ký vào chính nó không |
| `chronyc tracking` | Lệch giờ nhỏ | Lệch giờ gây lỗi SSL giữa host, Capsule, Satellite |

`satellite-maintain health --help` và `satellite-maintain service --help` liệt kê thêm tuỳ chọn.

## Dịch vụ

```bash
satellite-maintain service list
satellite-maintain service status
satellite-maintain service stop
satellite-maintain service start
satellite-maintain service restart
satellite-maintain service stop --exclude postgresql
```

| Dịch vụ | Vai trò | Khi hỏng thấy gì |
|---|---|---|
| `postgresql` | CSDL foreman, candlepin, pulpcore | `hammer ping` database FAIL, Web UI lỗi |
| `foreman`, `httpd` | Web UI, API, reverse proxy | Không vào được Web UI/API |
| `tomcat` | Candlepin (subscription, manifest) | `candlepin` FAIL; đăng ký host, refresh lỗi |
| `pulpcore-api`, `pulpcore-content`, `pulpcore-worker@N` | Lưu và phân phối content | Sync, publish lỗi; host không tải được gói |
| `dynflow-sidekiq@orchestrator`, `dynflow-sidekiq@worker-*` | Chạy task nền | Task đứng ở pending/paused |
| `redis` | Hàng đợi, cache cho task | Task không chạy |
| `foreman-proxy` | Smart proxy tích hợp (REX, OpenSCAP, đăng ký) | Capsule đỏ, job REX lỗi |

Danh sách trên lấy từ lab RH403; số `pulpcore-worker@N` tuỳ cấu hình. Nên dùng `satellite-maintain service` thay cho `systemctl` để dừng/khởi động đúng thứ tự cả nhóm. Kiến thức systemd chung: [systemd cơ bản](https://github.com/lovelybugf/kb-base-devops/blob/main/linux/systemd-basics.md).

## Log

| Thư mục / lệnh | Nội dung |
|---|---|
| `/var/log/foreman/production.log` | Web UI, API, task (đọc đầu tiên) |
| `/var/log/foreman-proxy/proxy.log` | Capsule/smart proxy: REX, OpenSCAP, đăng ký, shellhook |
| `/var/log/candlepin/candlepin.log`, `error.log`; `/var/log/tomcat/` | Subscription, manifest |
| `journalctl --unit 'pulpcore*'` | Pulp (ghi vào journal, không có file riêng) |
| `/var/log/foreman-installer/` | `satellite-installer`, `capsule-certs-generate` |
| `/var/log/foreman-maintain/` | `satellite-maintain`: backup, restore, health check, upgrade |
| `/var/log/httpd/`, `/var/log/messages` | Apache, hệ thống |
| `/var/log/rhsm/rhsm.log` (trên host) | subscription-manager của host |

- `foreman-tail` xem log Satellite theo thời gian thực.
- `sos report` gom cấu hình và log để mở case Red Hat; đã lọc mật khẩu/token nhưng vẫn chứa thông tin nhạy cảm → gửi thẳng cho người nhận, không đặt ở nơi công khai.
- Đổi sang journald: `satellite-installer --foreman-logging-type journald --foreman-proxy-log JOURNAL`, xem bằng `journalctl --unit foreman` hoặc `journalctl REQUEST=<REQUEST_ID>`. Trả về file: `--reset-foreman-logging-type --reset-foreman-proxy-log`. Log JSON: `--foreman-logging-layout json --foreman-logging-type file`.

## Tăng mức log khi debug

Mặc định INFO. Bật debug xong phải trả về mặc định để log không phình.

| Thành phần | Bật debug | Trả về mặc định |
|---|---|---|
| Tất cả (qua hammer) | `hammer admin logging --all --level-debug` rồi `satellite-maintain service restart` | `hammer admin logging --all --level-production` rồi restart |
| Một thành phần | `hammer admin logging --components <COMPONENT> --level-debug` (xem `hammer admin logging --list`) | như trên |
| Foreman | `satellite-installer --foreman-logging-level debug` | `satellite-installer --reset-foreman-logging-level` |
| Capsule | `satellite-installer --foreman-proxy-log-level DEBUG` | `satellite-installer --reset-foreman-proxy-log-level` |
| Candlepin | `satellite-installer --katello-candlepin-loggers log4j.logger.org.candlepin:DEBUG` | `satellite-installer --reset-katello-candlepin-loggers` |
| Logger riêng (sql, ldap, permissions...) | `satellite-installer --foreman-loggers ldap:true --foreman-loggers sql:true` | `satellite-installer --reset-foreman-loggers` |
| Hammer | `:log_level: 'debug'` trong `/etc/hammer/cli_config.yml`; log ở `~/.hammer/log/hammer.log` | sửa lại file |
| Redis | `loglevel debug` trong `/etc/redis/redis.conf`, `systemctl restart redis` | `satellite-installer` ghi đè về mặc định |
| Pulp | Khối `LOGGING` mức DEBUG trong `/etc/pulp/settings.py`, restart `pulpcore-api pulpcore-content pulpcore-worker@*` | `satellite-installer` ghi đè về mặc định |
| Installer | `satellite-installer --verbose-log-level debug` (chỉ ảnh hưởng màn hình) | — |

Logger `blob` ghi nội dung template đã render, có thể chứa dữ liệu nhạy cảm. Danh sách logger: `awk '/add_loggers/,/^$/' /usr/share/foreman/config/application.rb`. Mức log Red Hat Lightspeed và OpenVox: Phụ lục C.2, C.9–C.10.

## Task lỗi hoặc treo

- Xem ở Monitor > Satellite Tasks > Tasks: lọc theo trạng thái, xem chi tiết, **Cancel** hoặc **Resume** một hay nhiều task. Bằng CLI: `hammer task list --search 'state = paused'`, `hammer task info --id <TASK_ID>`.
- Task paused thường do một backend tạm mất (dịch vụ dừng, đĩa đầy, mạng tới Capsule): sửa nguyên nhân rồi Resume; health check có mục riêng cho task paused.
- Lỗi `There was an issue with the backend service candlepin: Connection refused` khi tạo task: Satellite kiểm tra backend trước khi chạy task và thấy dịch vụ chết → kiểm tra `hammer ping`, restart dịch vụ. Setting `check_services_before_actions` tắt được bước kiểm tra này, chỉ dùng khi chính bước kiểm tra gây lỗi.
- Task tới Capsule/host quá hạn trên đường truyền chậm: Administer > Settings, tìm `%_timeout`; tăng `%_finish_timeout` khi băng thông thấp, `%_accept_timeout` khi độ trễ cao (đơn vị giây).
- Dọn task cũ: xem [Quản lý dung lượng](storage-management.md).

## Sự cố storage: Web UI 500, SSH bị ngắt (lab, đã kiểm chứng)

| | |
|---|---|
| Triệu chứng | Web UI trả HTTP 500; SSH vào Satellite bị reset (`kex_exchange_identification: Connection reset`); console VM báo I/O timeout trên `sda`/`sdb`, XFS tự shutdown |
| Nguyên nhân | Một host ESXi chuyển Not responding làm VM mất truy cập datastore vSAN |
| Xử lý | Xem console VM ở chế độ chỉ đọc trước. Chờ host ESXi về Connected và vSAN health sạch, rồi mới **Reset** VM. Không dùng Restart Guest OS (máy đã treo, guest không phản hồi); không Reset khi vSAN chưa ổn định |
| Sau khi lên | XFS tự replay log trên các LV (có cả LV của `/var/lib/pulp` và `/var/lib/pgsql`); `hammer ping` đủ 9 mục ok; `satellite-maintain health check` OK |

Sau sự cố kiểu này nên kiểm tra thêm task paused, trạng thái Capsule và chạy một bản backup full mới. Trạng thái Global "Error" của host sau sự cố có thể chỉ do còn errata bảo mật áp dụng được, không phải hậu quả của sự cố.

## Triệu chứng thường gặp khác

| Triệu chứng | Nguyên nhân hay gặp | Xử lý |
|---|---|---|
| Sync/publish lỗi, task Pulp fail | `/var/lib/pulp` đầy | [Quản lý dung lượng](storage-management.md), chạy lại task |
| Capsule hiện biểu tượng đỏ ở Infrastructure > Capsules | Dịch vụ `foreman-proxy` trên Capsule dừng, chặn cổng 9090 | Restart `foreman-proxy` trên Capsule, kiểm tra tường lửa, chọn Refresh |
| Lỗi SSL giữa host/Capsule/Satellite | Lệch giờ, sai CA | `chronyc tracking`; xem [Chứng chỉ](../installation/certificates.md) |
| Health check cảnh báo không mở được log của `sssd_kcm` | Thư mục `/var/log/sssd` chưa có | `mkdir -p /var/log/sssd; chmod 750 /var/log/sssd; systemctl restart sssd-kcm` |
| `dnf install` trên Satellite/Capsule bị chặn | Khoá gói (foreman-protector) mặc định bật | `satellite-maintain packages install <PKG>` (tự chạy lại installer, có thể restart dịch vụ) |
| Test email không tới, `mailq` báo `Mail system is down` | Postfix chưa chạy | `systemctl start postfix`; xem [Thông báo và webhook](notifications-and-webhooks.md) |
| Dịch vụ bị dừng sau khi cài agent/hardening | Agent chặn tiến trình, thư mục, cổng | `rpm -qa --last \| head`, log agent, áp danh sách loại trừ |

## Liên quan

- [Giám sát và chỉ số](monitoring-and-metrics.md)
- [Sao lưu và khôi phục](backup-and-restore.md)
- [Quản lý dung lượng lưu trữ](storage-management.md)
- [Thành phần và dịch vụ](../architecture/components-and-services.md)
- [Cổng mạng](../architecture/network-ports.md)
- [Hammer cheatsheet](../ansible/hammer-cheatsheet.md)

## Nguồn

- Red Hat Satellite 6.19, Administering Red Hat Satellite, Phụ lục A Starting and stopping Satellite services (PDF tr. 118), Phụ lục B Logging and reporting problems (PDF tr. 119–125), Phụ lục C Increasing logging levels (PDF tr. 126–130), Phụ lục E packages (PDF tr. 132), mục 4.2.3 (PDF tr. 25), mục 18.2 Satellite task management (PDF tr. 101–102) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Khoá RH403, chương 11: ví dụ đầu ra `hammer ping`, `satellite-maintain health check`, danh sách dịch vụ.
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: `service status -b`, `hammer ping`, health check trên Satellite và Capsule đạt trong UAT; bảng sự cố thường gặp.
- Lab Satellite 6.19 (ẩn danh), 2026: sự cố mất vSAN, xử lý bằng Reset VM sau khi host ESXi và vSAN ổn định; XFS tự phục hồi, `hammer ping` và health check OK.
