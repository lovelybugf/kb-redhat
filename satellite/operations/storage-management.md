---
title: Quản lý và giảm dung lượng lưu trữ của Satellite
tags: [howto, satellite, storage, pulp, content-view, lvm]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Administering Red Hat Satellite, chương 15 (PDF tr. 84–88); Managing content, mục 9.5 (PDF tr. 84–87); Installing Satellite Server in a connected network environment, mục 1.3–1.4 (PDF tr. 9–11); RH403 chương 11"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Dung lượng Satellite dồn vào `/var/lib/pulp` (content) và `/var/lib/pgsql` (CSDL); theo dõi hằng tuần và cảnh báo ở khoảng 80 %. Giảm bằng cách xoá Content View version cũ rồi dọn orphan, Reclaim Space cho repo on_demand, dọn audit/report/task cũ, `vacuumdb --full` cho PostgreSQL; đĩa đầy thì nới LV.

## Thư mục cần theo dõi

| Thư mục | Lúc cài | Khi chạy (tham khảo của hãng) | Chứa gì |
|---|---|---|---|
| `/var/lib/pulp` | 1 MB | 300 GB | Gói, metadata, export của repo và CV version |
| `/var/lib/pgsql` | 1 GB | 20 GB | CSDL foreman, candlepin, pulpcore |
| `/var/log` | 10 MB | 10 GB | Log, xoay vòng bằng logrotate |
| `/var/lib/containers` (nếu dùng Red Hat Lightspeed trong Satellite) | 20 GB | 30 GB | Container image của Lightspeed |
| `/usr` | 10 GB | — | Phần mềm |
| `/opt/puppetlabs` | 500 MB | — | OpenVox/Puppet |

Con số runtime đo với repo RHEL 7, 8, 9 đã sync; thực tế tuỳ số repo, download policy và số CV version.

```bash
df -hT /var/lib/pulp /var/lib/pgsql /var/log
du -sh /var/lib/pulp /var/lib/pgsql/data
```

- Web UI: Infrastructure > Capsules > tên Capsule > Overview hiển thị dung lượng Pulp đã dùng và còn trống trên Capsule.
- Thực hành tốt của hãng: `/var` trên LVM để mở rộng dễ; storage băng thông cao, độ trễ thấp cho `/var/lib/pulp` và `/var/lib/pgsql`; không dùng GFS2; `/var/lib/pulp` không được là symlink.
- `/var/lib/pulp` trên NFS phải gắn SELinux context trong `/etc/fstab`, sau đó `restorecon -R /var/lib/pulp`:

```bash
<NFS_SERVER>:<EXPORT>  /var/lib/pulp  nfs  context="system_u:object_r:pulpcore_var_lib_t:s0"  1 2
```

- `/tmp` tách riêng thì phải mount `exec` (puppetserver cần).
- Thư mục chứa bản sao lưu nên là đĩa/NFS riêng, không chung LV với Pulp (xem [Sao lưu và khôi phục](backup-and-restore.md)).

## Dọn content

### Xoá Content View version cũ

Mặc định Satellite giữ mọi version. Chỉ xoá được version không còn nằm trong lifecycle environment nào; host hoặc Activation Key đang dùng environment bị ảnh hưởng phải được chuyển trước.

- Web UI: Content > Lifecycle > Content Views > chọn CV > tab Versions > tích các version > menu tuỳ chọn > Delete (wizard hiển thị environment bị ảnh hưởng).
- Hammer:

```bash
hammer content-view info --name "<CONTENT_VIEW>" --organization-id <ORG_ID>
hammer content-view remove --id <CV_ID> --content-view-version-ids <VERSION_ID_1>,<VERSION_ID_2>
# hoặc giữ N version gần nhất, xoá phần còn lại không thuộc environment nào
hammer content-view purge --name "<CONTENT_VIEW>" --organization-id <ORG_ID> --versions-to-keep <N>
```

Đánh đổi: ít version hơn thì tiết kiệm đĩa nhưng mất khả năng quay về snapshot content cũ. Ansible: module `redhat.satellite.content_view_version`.

### Dọn orphaned content

Xoá CV version hoặc repo chỉ gỡ liên kết; dữ liệu Pulp không còn ai dùng (orphan) được dọn bằng tác vụ chạy hằng tuần, hoặc chạy tay bằng root:

```bash
foreman-rake katello:delete_orphaned_content RAILS_ENV=production
```

### Reclaim Space cho repo on_demand

Repo `on_demand` lưu lại gói sau lần đầu host yêu cầu. Reclaim Space xoá các gói đã tải này khỏi đĩa (host không bị gỡ gói; lần sau có host cần thì tải lại):

- Một repo: Content > Products > chọn product > tab Repositories > chọn repo > Select Actions > Reclaim Space.
- Nhiều repo: tích các repo > Reclaim Space.
- Capsule: Infrastructure > Capsules > chọn Capsule > Reclaim space.

Reclaim chỉ áp dụng cho repo còn tồn tại; repo đã xoá thì dựa vào dọn orphan. Đổi repo từ `immediate` sang `on_demand` rồi reclaim giải phóng được nhiều nhất, nhưng xem điều kiện mạng ở [Download policy](../content/download-policy.md) trước khi đổi.

## Dọn bản ghi trong CSDL

Chạy bằng root:

```bash
# Audit cũ hơn N ngày (mặc định 90)
foreman-rake audits:expire days=<N>
# Config report cũ hơn N ngày (mặc định 90)
foreman-rake reports:expire days=<N>
# Task: ví dụ xoá task sync repo đã kết thúc
foreman-rake foreman_tasks:cleanup TASK_SEARCH='label = Actions::Katello::Repository::Sync' STATES='stopped'
# Xoá một task theo ID (ví dụ lỡ đưa dữ liệu nhạy cảm vào task)
foreman-rake foreman_tasks:cleanup TASK_SEARCH="id=<TASK_ID>"
```

- Audit tự xoá: Administer > Settings > General > **Saved audits interval** (số ngày; để trống là không tự xoá).
- `foreman-rake audits:anonymize days=<N>` chỉ xoá thông tin user/IP trong audit, không giảm dung lượng.
- Task: mặc định có cron dọn hằng ngày lúc 19:45, xoá task thành công cũ hơn 30 ngày và mọi task cũ hơn 1 năm. Đổi giờ: `satellite-installer --foreman-plugin-tasks-cron-line "00 15 * * *"`; tắt/bật: `--foreman-plugin-tasks-automatic-cleanup false|true`; đổi quy tắc tuổi trong mục `:rules:` của `/etc/foreman/plugins/foreman-tasks.yaml`. Satellite nâng cấp từ bản cũ nên kiểm tra cron này đã bật.
- Có thể đặt các lệnh `expire` vào cron theo chu kỳ mong muốn.

## Thu hồi dung lượng PostgreSQL

Bản ghi đã xoá/cập nhật vẫn chiếm chỗ đến khi vacuum. `vacuumdb --full` cần dừng các dịch vụ khác nên làm trong khung bảo trì. Chạy bằng root:

```bash
satellite-maintain service stop --exclude postgresql
su - postgres -c 'vacuumdb --full --all'
satellite-maintain service start
```

## Khi đĩa Pulp đầy

1. Để các task Pulp đang chạy kết thúc, **không** khởi chạy task mới (sẽ lỗi vì thiếu chỗ).
2. Giải phóng hoặc nới dung lượng:
   - dọn orphan (thường giải phóng ít vì đã chạy hằng tuần);
   - chuyển repo `immediate` sang `on_demand` rồi Reclaim Space;
   - xoá CV version cũ rồi dọn orphan;
   - nới LV chứa `/var/lib/pulp`. XFS/ext4 nới được khi đang mount, ví dụ `lvextend -r -L +<SIZE>G /dev/<VG>/<LV>`; chi tiết: [LVM](https://github.com/lovelybugf/kb-base-devops/blob/main/linux/storage-lvm.md). Filesystem không nới nóng được thì `satellite-maintain service stop`, nới, rồi `satellite-maintain service start`.
3. Chạy lại các task Pulp đã lỗi vì đầy đĩa.

## Lưu ý

- Xoá CV version không giảm `df` ngay: phải đợi (hoặc chạy) dọn orphan.
- `immediate` cho repo lớn làm `/var/lib/pulp` tăng mạnh; ước lượng trước khi đổi policy hoặc bật repo mới.
- Export content (Inter-Satellite Sync) ghi vào `/var/lib/pulp/exports`; xoá export cũ sau khi đã chuyển đi.
- Trong triển khai thực tế: kiểm tra `df` hằng tuần, cảnh báo ở 80 % cho Pulp/CSDL/log, nơi lưu backup giữ dưới 70 %.

## Liên quan

- [Download policy](../content/download-policy.md)
- [Content View và lifecycle](../content/content-views-and-lifecycle.md)
- [Inter-Satellite Sync](../content/inter-satellite-sync.md)
- [Sizing và tuning](../architecture/sizing-and-tuning.md)
- [Giám sát và chỉ số](monitoring-and-metrics.md)
- [Health check và xử lý sự cố](health-check-and-troubleshooting.md)
- [LVM (kb-base-devops)](https://github.com/lovelybugf/kb-base-devops/blob/main/linux/storage-lvm.md)

## Nguồn

- Red Hat Satellite 6.19, Administering Red Hat Satellite, chương 15 Reducing storage use on Satellite Server (PDF tr. 84–88), Phụ lục D Anonymizing audit records (PDF tr. 131) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing content, mục 9.5 Managing content view versions (PDF tr. 84–87) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Installing Satellite Server in a connected network environment, mục 1.3 Storage requirements, 1.4 Best practices for optimizing storage (PDF tr. 9–11) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Khoá RH403, chương 11: giải thích vacuum PostgreSQL, reclaim space, dọn audit/report/task.
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: ngưỡng cảnh báo dung lượng, nới LV + XFS nóng bằng `lvextend -r`.
