# Quản lý host

> Đăng ký host, host group và tham số, remote execution, vá errata, gói, nâng cấp RHEL, report template, Lightspeed.

## Trang

<!-- AUTO-INDEX START -->
- [Activation Key chỉ áp dụng lúc đăng ký host](activation-key-registration-only.md) — Activation Key (AK) chỉ được áp dụng một lần, lúc đăng ký host; sửa AK sau đó không làm thay đổi các host đã đăng ký.
- [Tắt Red Hat Lightspeed (Insights) trên Satellite và host](disable-lightspeed.md) — Đặt global parameter boolean `host_registration_insights` và `host_registration_insights_inventory` bằng `false` để host đăng ký mới không cài và không gửi dữ liệu Lightspeed.
- [Huỷ đăng ký và xoá host khỏi Satellite (unregister, delete, VM)](host-deletion-and-unregister.md) — `subscription-manager unregister` chỉ huỷ đăng ký; bản ghi host vẫn nằm lại trên Satellite trừ khi bật setting toàn hệ thống Delete Host upon unregister (`unregister_delete_host`, mặc định No).
- [Host group, tham số host, host collection và system purpose](host-groups-and-parameters.md) — Host group là khuôn thiết lập dùng chung (OS, LE/CV, Capsule, Ansible role, tham số), lồng nhau được và nhóm con ghi đè nhóm cha; host collection chỉ là tập host để thao tác hàng loạt.
- [Đăng ký host lên Satellite bằng global registration](host-registration.md) — Hosts → Register Host (hoặc `hammer host-registration generate-command`) sinh một lệnh đăng ký dạng curl có token JWT; chạy lệnh đó bằng root trên host để đăng ký qua Satellite hoặc Capsule với Activation Key đã chọn.
- [Huỷ đăng ký và đăng ký lại host lên Satellite (global registration)](host-reregistration.md) — `subscription-manager unregister` và `clean` không đụng tới `/etc/rhsm/rhsm.conf`, nên host vẫn trỏ về Satellite cũ; đừng xoá `rhsm.conf` mà để lệnh global registration tự cấu hình lại, chỉ cần dọn CA cũ và proxy cũ.
- [Cài, gỡ, cập nhật gói từ Satellite và dùng Tracer](packages-and-tracer.md) — Mọi thao tác gói trên host từ Satellite (cài, nâng cấp, gỡ, bật/tắt repository) đều chạy thành job remote execution, và chỉ dùng được gói có trong Content View environment của host.
- [Runbook vá host bằng errata (REX, Ansible, chu kỳ và vá khẩn)](patching-with-errata.md) — Vá host trên Satellite: xác định errata Applicable/Installable, đưa bản vá vào Content View của môi trường, áp dụng qua remote execution (Script hoặc Ansible), khởi động lại nếu cần rồi kiểm tra và báo cáo.
- [Remote execution (REX) qua SSH và pull mode](remote-execution.md) — Remote execution (REX) chạy job template (Script hoặc Ansible) trên host thông qua Capsule, mặc định đẩy qua SSH bằng khoá của user `foreman-proxy`, hoặc để host tự kéo job qua MQTT (pull mode, chỉ Script).
- [Report template và cảnh báo CVE theo ngưỡng](report-templates-and-cve-alert.md) — Report template là đoạn ERB chạy trên Satellite, xuất dữ liệu host/errata ra CSV hoặc JSON/YAML/HTML, sinh từ Web UI hay `hammer report-template generate`, chạy định kỳ thì dùng cron.
- [Nâng cấp RHEL lên major tiếp theo bằng Leapp qua Satellite (kèm nâng minor)](rhel-major-upgrade-leapp.md) — Satellite nâng cấp tại chỗ RHEL 7→8, 8→9, 9→10 bằng hai job REX: Preupgrade check with Leapp (xem báo cáo, xử lý mọi Inhibitor) rồi Upgrade with Leapp.
<!-- AUTO-INDEX END -->
