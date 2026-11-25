# Quản lý nội dung

> Subscription/manifest, repository và đồng bộ, download policy, content view và lifecycle environment, errata, đồng bộ giữa các Satellite, virt-who.

## Trang

<!-- AUTO-INDEX START -->
- [Alternate content source và tối ưu đồng bộ, lưu trữ (mirroring policy)](alternate-content-sources.md) — Alternate content source (ACS) cho Satellite/Capsule tải gói từ một nguồn gần hơn (thư mục cục bộ, mirror nội bộ, RHUI hoặc thẳng Red Hat CDN) trong khi metadata vẫn lấy từ upstream; tạo xong phải Refresh.
- [Content View, Lifecycle Environment và content view environment](content-views-and-lifecycle.md) — Content View (CV) là snapshot có phiên bản của repository trong Library; publish tạo version mới, promote đưa version qua từng Lifecycle Environment (LE) như Library → DEV → PROD.
- [Download policy của repository (on_demand và immediate)](download-policy.md) — Mặc định repo Red Hat dùng `on_demand` (chỉ tải metadata, gói được tải khi host/Capsule yêu cầu), repo custom dùng `immediate`.
- [Errata trong Content View, incremental update và quy trình vá khẩn](errata-and-incremental-update.md) — Errata mới sync chỉ nằm trong Library nên trên host là Applicable; muốn Installable phải publish version mới hoặc incremental update rồi promote tới LE của host và chờ Capsule đồng bộ.
- [Đồng bộ nội dung giữa các Satellite (ISS) bằng mạng hoặc export/import](inter-satellite-sync.md) — Satellite disconnected nhận nội dung từ Satellite upstream có Internet bằng Network Sync (kéo qua HTTPS) hoặc Export Sync cho máy air-gapped (export, chép đĩa, import).
- [Nội dung ngoài RPM: container, Flatpak, ISO, Ansible collection, file](other-content-types.md) — Ngoài RPM, Satellite quản lý container image (Satellite làm registry, image đi qua CV/LE), Flatpak (kiểu docker), ISO và file tùy ý (repository loại file), Ansible collection (Satellite làm Galaxy server).
- [Repository Red Hat, custom repository và đồng bộ nội dung](repositories-and-sync.md) — Bật repository Red Hat theo phụ lục A (BaseOS + AppStream đúng bản RHEL, thêm Kickstart nếu provisioning), tạo product riêng cho nội dung ngoài Red Hat kèm GPG key, rồi gắn sync plan chạy hằng ngày.
- [Subscription manifest, Simple Content Access và subscription allocation](subscriptions-manifest.md) — Mỗi Organization trong Satellite cần đúng một subscription manifest (một subscription allocation) để lấy chứng chỉ truy cập Red Hat CDN và biết được phép bật repository nào.
- [virt-who cho subscription máy ảo: cấu hình, deploy và xử lý sự cố](virt-who.md) — virt-who đọc danh sách hypervisor và máy ảo từ nền tảng ảo hóa rồi báo về Satellite, để subscription tính theo host (ví dụ RHEL for Virtual Datacenters) được đối soát đúng.
<!-- AUTO-INDEX END -->
