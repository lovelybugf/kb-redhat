# Vận hành

> Sao lưu và khôi phục, health check và xử lý sự cố, cứu host khi Satellite hỏng, dung lượng, thông báo và webhook, giám sát.

## Trang

<!-- AUTO-INDEX START -->
- [Sao lưu và khôi phục Satellite, Capsule bằng satellite-maintain](backup-and-restore.md) — Cách sao lưu chuẩn là `satellite-maintain backup offline` kèm Pulp content: full hằng tuần, incremental hằng ngày, rồi chép sang nơi lưu riêng được bảo vệ.
- [Cứu host RHEL khi Satellite hỏng](client-rescue-when-satellite-down.md) — Satellite hỏng không làm host ngừng chạy; host đăng ký qua Capsule còn sống vẫn `dnf install` được, chỉ refresh, đăng ký mới và job từ Satellite là lỗi.
- [Khắc phục thảm hoạ, clone và đổi tên Satellite](disaster-recovery-and-cloning.md) — Red Hat đưa ra bốn mô hình DR: snapshot VM, active/passive dùng chung storage, active/passive bằng backup/restore, và hai Satellite cùng active.
- [Health check, dịch vụ, log và xử lý sự cố Satellite](health-check-and-troubleshooting.md) — Khi Satellite có vấn đề, chạy theo thứ tự `satellite-maintain service status -b`, `hammer ping`, `satellite-maintain health check`, rồi xem task paused/error và log.
- [Giám sát Satellite, Capsule và usage metrics](monitoring-and-metrics.md) — Giám sát trên Web UI qua Monitor > Dashboard, Satellite Tasks, Audits và trang Capsules; phía máy chủ dựa vào `satellite-maintain health check`, `hammer ping` và dung lượng đĩa.
- [Email thông báo, relay thư ra ngoài và webhook của Satellite](notifications-and-webhooks.md) — Satellite không gửi email mặc định: cấu hình SMTP ở Administer > Settings > Email, rồi bật Mail Enabled và chọn loại thông báo cho từng user.
- [Quản lý và giảm dung lượng lưu trữ của Satellite](storage-management.md) — Dung lượng Satellite dồn vào `/var/lib/pulp` (content) và `/var/lib/pgsql` (CSDL); theo dõi hằng tuần và cảnh báo ở khoảng 80 %.
<!-- AUTO-INDEX END -->
