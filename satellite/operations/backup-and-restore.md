---
title: Sao lưu và khôi phục Satellite, Capsule bằng satellite-maintain
tags: [runbook, satellite, backup, capsule, pulp]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Administering Red Hat Satellite, chương 11–12 (PDF tr. 62–71), mục 4.4.4 (PDF tr. 29–30); RH403 chương 11; kinh nghiệm triển khai thực tế (ẩn danh)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Cách sao lưu chuẩn là `satellite-maintain backup offline` kèm Pulp content: full hằng tuần, incremental hằng ngày, rồi chép sang nơi lưu riêng được bảo vệ. Khôi phục bằng `satellite-maintain restore` trên máy cùng hostname, cùng phiên bản và trạng thái FIPS, full trước rồi từng incremental đúng thứ tự.

## Chọn phương thức

| Phương thức | Lệnh | Dịch vụ trong lúc chạy | Dùng khi |
|---|---|---|---|
| Offline full | `satellite-maintain backup offline <BACKUP_DIR>` | Dừng toàn bộ; chế độ bảo trì, tường lửa chặn 443 từ ngoài | Mặc định cho Satellite và Capsule |
| Offline incremental | `... offline --incremental <PREV_BACKUP> <BACKUP_DIR>` | Dừng toàn bộ, ngắn hơn full | Các ngày giữa hai bản full |
| Online | `satellite-maintain backup online <BACKUP_DIR>` | Đa số dịch vụ chạy; background worker dừng, task cần worker nằm ở pending | Không được dừng dịch vụ; chậm hơn, Pulp thay đổi giữa chừng thì chạy lại từ đầu |
| Bỏ Pulp content | `... offline --skip-pulp-content <BACKUP_DIR>` | Dừng toàn bộ | Debug, lấy file cấu hình; **không** dùng để khôi phục Satellite production |
| Snapshot VM / sao lưu thông thường | `satellite-maintain service stop` → chụp → `satellite-maintain service start` | Dừng toàn bộ | Có hypervisor hoặc công cụ sao lưu của tổ chức |

- Bộ tài liệu 6.19 không còn `satellite-maintain backup snapshot` (snapshot LVM) và không còn tuỳ chọn `--whitelist` cho lệnh backup (`--whitelist` chỉ còn ở `satellite-maintain upgrade run`). Tài liệu nội bộ cũ còn chép hai lệnh này thì bỏ.
- Khoá RH403 (viết cho 6.15) khuyên chỉ dùng online backup để debug; tài liệu 6.19 giới thiệu online backup cho trường hợp cần giữ dịch vụ. Với lịch định kỳ, offline vẫn là lựa chọn an toàn và nhanh hơn.
- Snapshot VM chỉ nhất quán khi đã dừng dịch vụ trước lúc chụp.

## Chuẩn bị

### Ước lượng dung lượng

```bash
du -sh /var/lib/pgsql/data /var/lib/pulp
du -csh /var/lib/tftpboot /etc /root/ssl-build /var/www/html/pub /opt/puppetlabs
```

| Dữ liệu | Tỷ lệ nén | Ví dụ trong tài liệu hãng |
|---|---|---|
| PostgreSQL `/var/lib/pgsql/data` | 80–85 % | 100 GB → 20 GB |
| Pulp `/var/lib/pulp` | không nén | 100 GB → 100 GB |
| Cấu hình (`/etc`, `/root/ssl-build`, `/var/lib/tftpboot`, `/var/www/html/pub`, `/opt/puppetlabs`) | 85 % | 942 MB → 141 MB |

Dung lượng cần = dữ liệu chưa nén + dữ liệu đã nén + 20 % dự phòng. Ví dụ trên: 201 GB + 120 GB = 321 GB, thêm 20 % → cấp khoảng 385 GB. Bản sao lưu không chứa container image của Red Hat Lightspeed trong Satellite.

### Nơi lưu và bảo mật

- Lưu trên thiết bị riêng, hệ thống riêng; không đặt chung LV với `/var/lib/pulp`.
- NFS: user root của Satellite/Capsule phải ghi được, nên export không được có `root_squash`/`all_squash`.
- Bản sao lưu chứa `/root/ssl-build` (khoá, chứng chỉ, file request, hostname) và toàn bộ CSDL → mã hoá hoặc giới hạn quyền, không để Root CA private key ở cùng chỗ.

### Khung giờ

- Báo người dùng lưu thay đổi; không để trùng sync plan, publish/promote, job REX, lịch quét OpenSCAP. Kiểm tra task đang chạy:

```bash
hammer task list --search "state = running"
```

- Capsule đứng sau load balancer: sao lưu lần lượt từng Capsule, lệch giờ nhau. Trong triển khai thực tế, host vẫn `dnf` được qua VIP khi một Capsule đang sao lưu offline.

## Sao lưu

Các lệnh dưới chạy bằng root.

### Full

```bash
# Trên Satellite
satellite-maintain backup offline --assumeyes /var/satellite-backup
# Trên Capsule
satellite-maintain backup offline --assumeyes /var/foreman-proxy-backup
```

- Mỗi lần chạy tạo một thư mục con có dấu thời gian (trên Satellite: `satellite-backup-YYYY-MM-DD-HH-MM-SS`), không ghi đè bản cũ; dịch vụ tự khởi động lại khi xong.
- `--assumeyes` bỏ câu hỏi xác nhận (bắt buộc khi chạy bằng cron). `--preserve-directory` ghi thẳng vào thư mục chỉ định và không xoá dữ liệu nếu backup lỗi.

### Incremental

```bash
satellite-maintain backup offline --assumeyes --incremental <BACKUP_DIR>/<FULL_BACKUP> <BACKUP_DIR>
satellite-maintain backup offline --assumeyes --incremental <BACKUP_DIR>/<INCREMENTAL_1> <BACKUP_DIR>
```

Phải có một bản full làm gốc; mỗi incremental trỏ vào bản liền trước. Muốn mở nhánh mới thì trỏ lại vào bản full. Khi khôi phục cần bản full và **toàn bộ** chuỗi incremental của nhánh đó.

### Lịch tự động: full Chủ nhật, incremental các ngày khác

Dựa trên script mẫu của Red Hat (mục 4.4.4), thêm kiểm tra mount và xoá bản cũ; dạng này đã chạy theo cron trong triển khai thực tế. Chạy bằng root:

```bash
cat > /usr/local/sbin/satellite-backup.sh << 'EOF'
#!/bin/bash -e
PATH=/sbin:/bin:/usr/sbin:/usr/bin
BACKUP_MNT=<BACKUP_MNT>
DEST=$BACKUP_MNT/$(hostname -s)
findmnt "$BACKUP_MNT" >/dev/null || { echo "$(date) chua mount $BACKUP_MNT"; exit 1; }
mkdir -p "$DEST"; chmod 755 "$DEST"
LAST=$(ls -td -- "$DEST"/satellite-backup-*/ 2>/dev/null | head -n 1)
if [[ $(date +%w) == 0 || -z "$LAST" ]]; then
  echo "$(date) FULL"
  satellite-maintain backup offline --assumeyes "$DEST"
  find "$DEST" -maxdepth 1 -type d -name 'satellite-backup-*' -mtime +13 -exec rm -rf {} +
else
  echo "$(date) INCREMENTAL tu $LAST"
  satellite-maintain backup offline --assumeyes --incremental "$LAST" "$DEST"
fi
echo "$(date) Hoan tat"
EOF
chmod 700 /usr/local/sbin/satellite-backup.sh
echo '0 3 * * * root /usr/local/sbin/satellite-backup.sh >> /var/log/satellite-backup.log 2>&1' > /etc/cron.d/satellite-backup
```

- `#!/bin/bash -e` phải là dòng đầu; `PATH` phải có `/sbin` và `/usr/sbin`, nếu không `satellite-maintain` lỗi khi chạy từ cron.
- `-mtime +13` sau mỗi bản full giữ lại khoảng hai chuỗi. Giờ chạy (ví dụ 03:00) phải tránh sync plan và lịch quét của host.
- `findmnt` chặn trường hợp NFS chưa mount làm backup ghi tràn vào `/`.

## Kiểm tra bản sao lưu

Chỉ coi là có backup khi đã restore thử thành công (mục Khôi phục); kiểm tra hằng ngày dưới đây chỉ phát hiện lỗi sớm.

```bash
ls -lh <BACKUP_DIR>/<BACKUP>/
head <BACKUP_DIR>/<BACKUP>/metadata.yml
sudo -u postgres head -c 1 <BACKUP_DIR>/<BACKUP>/foreman.dump > /dev/null && echo "postgres doc duoc"
tail -n 20 /var/log/satellite-backup.log
hammer ping
```

Bản full kèm Pulp thường có `metadata.yml`, `config_files.tar.gz`, `candlepin.dump`, `foreman.dump`, `pulpcore.dump`, `pulp_data.tar` (quan sát thực tế, tài liệu hãng không liệt kê). Kiểm tra định kỳ: log mỗi ngày có một dòng hoàn tất, dung lượng nơi lưu chưa vượt ngưỡng.

## Khôi phục

Điều kiện: máy đích cùng hostname, cùng cấu hình, cùng phiên bản X.Y; máy gốc bật FIPS thì máy đích cũng bật; thư mục đích (đọc từ file cấu hình trong bản sao lưu) tồn tại và ghi được. Toàn bộ dữ liệu có trong bản sao lưu trên máy đích sẽ bị ghi đè.

Các bước, chạy bằng root:

1. Máy mới: cài RHEL và Satellite (hoặc Capsule) cùng phiên bản theo tài liệu cài đặt; dùng Red Hat Lightspeed trong Satellite thì bật lại.
2. Chép bản sao lưu về đĩa cục bộ (`/var/` hoặc `/var/tmp/`), kiểm tra dung lượng:

```bash
du -sh <BACKUP_DIR>
df -h <BACKUP_DIR>
restorecon -Rv /
```

3. Khôi phục bản full, sau đó từng incremental theo đúng thứ tự tạo:

```bash
satellite-maintain restore --assumeyes <BACKUP_DIR>/<FULL_BACKUP>
satellite-maintain restore --assumeyes <BACKUP_DIR>/<INCREMENTAL_1>
satellite-maintain restore --assumeyes <BACKUP_DIR>/<INCREMENTAL_2>
```

4. Kiểm chứng: `hammer ping`, `satellite-maintain health check`, `satellite-maintain service status`; trên Web UI dữ liệu đúng thời điểm sao lưu (cách thử: tạo một user trước khi backup, xoá sau khi backup, restore xong user phải có lại).
5. Sau khôi phục: Infrastructure > Capsules > từng Capsule > Synchronize (Complete Sync); chạy ngay một bản full mới vì chuỗi incremental cũ không còn khớp; máy mới thay máy cũ thì loại bỏ máy cũ, không chạy song song.

Log khi lỗi: `/var/log/foreman/production.log`, `/var/log/messages`, `/var/log/foreman-maintain/`. Trong triển khai thực tế, restore trực tiếp từ thư mục NFS đã mount vẫn thành công khi thư mục để quyền 755; tài liệu hãng yêu cầu chép về đĩa cục bộ, nên làm theo hãng.

### Khôi phục Capsule

- Bản sao lưu có Pulp: restore như Satellite rồi Optimized Sync.
- Không có Pulp hoặc mất máy: khôi phục từ snapshot VM (hãng khuyên chụp hằng tuần), hoặc cài Capsule mới **cùng hostname** bằng gói chứng chỉ `<CAPSULE_FQDN>-certs.tar` còn trên Satellite (hoặc tạo lại bằng `capsule-certs-generate`), đến khi Capsule kết nối được Satellite, rồi đồng bộ đầy đủ:

```bash
hammer capsule content synchronize --id <CAPSULE_ID> --skip-metadata-check true
```

## Chính sách mẫu (rút từ triển khai thực tế)

| Hạng mục | Giá trị đã áp dụng |
|---|---|
| Kiểu | Offline; Satellite luôn kèm Pulp content |
| Lịch | Full Chủ nhật, incremental các ngày khác; ngoài giờ sync plan; thêm một bản full trước mọi lần update/upgrade |
| Nơi lưu | Ổ riêng hoặc NFS riêng theo từng máy; giữ 1–2 tuần (khoảng hai chuỗi) |
| Capsule | Hai lựa chọn: offline kèm Pulp (khôi phục nhanh, tốn chỗ) hoặc hằng tuần `--skip-pulp-content` (gọn, khôi phục = cấu hình + Complete Sync từ Satellite) |
| Theo dõi | Log script mỗi ngày; dung lượng nơi lưu cảnh báo ở 70–80 % |
| Diễn tập restore | 6–12 tháng/lần và trước nâng cấp lớn, trên VM cách ly mạng, cùng hostname, cả full và ít nhất một incremental |

Lựa chọn `--skip-pulp-content` cho Capsule là quyết định của dự án (Satellite giữ đủ content để đồng bộ lại); hãng chỉ cảnh báo không dùng bản bỏ Pulp để khôi phục production, nên cần chấp nhận thời gian Complete Sync khi khôi phục.

## Lỗi hay gặp

| Triệu chứng | Nguyên nhân | Xử lý |
|---|---|---|
| Backup lỗi quyền khi ghi NFS | Export có `root_squash`/`all_squash` | Export cho phép root ghi |
| Backup làm đầy `/` | NFS chưa mount, lệnh ghi vào thư mục trống trên đĩa gốc | Kiểm tra `findmnt` trong script |
| Restore báo `Validate permissions for PostgreSQL dumps: FAIL` | Thư mục backup quyền 700, user postgres không đọc được dump | `chmod 755` thư mục chứa bản sao lưu |
| Restore từ chối chạy | Khác hostname, khác phiên bản X.Y hoặc khác FIPS | Dựng máy đích đúng điều kiện |
| Thiếu dữ liệu sau restore incremental | Sai thứ tự hoặc thiếu một mắt xích | Restore lại full rồi từng incremental theo thứ tự |
| Online backup kéo dài | Pulp thay đổi trong lúc chạy khiến quá trình chạy lại | Không sync/publish khi backup, hoặc dùng offline |
| Host lỗi tải gói sau restore | Capsule lệch nội dung so với Satellite | Complete Sync từng Capsule |

## Liên quan

- [Khắc phục thảm hoạ và clone Satellite](disaster-recovery-and-cloning.md)
- [Health check và xử lý sự cố](health-check-and-troubleshooting.md)
- [Quản lý dung lượng lưu trữ](storage-management.md)
- [Cứu host khi Satellite hỏng](client-rescue-when-satellite-down.md)
- [Nâng cấp Satellite](../installation/upgrade-satellite.md)
- [Capsule sau load balancer](../installation/capsule-load-balancer.md)

## Nguồn

- Red Hat Satellite 6.19, Administering Red Hat Satellite, chương 11 Backing up Satellite Server and Capsule Server (PDF tr. 62–68), chương 12 Restoring Satellite Server or Capsule Server from a backup (PDF tr. 69–71), mục 4.4.4 script mẫu (PDF tr. 29–30) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing content, mục đồng bộ Capsule `--skip-metadata-check` — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Khoá RH403, chương 11 Maintain a Red Hat Satellite Server: giải thích offline/online, incremental, thứ tự restore.
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: sao lưu offline Satellite và Capsule vào NFS theo cron, Capsule sao lưu lần lượt sau load balancer, restore thử thành công (dữ liệu về đúng thời điểm, `hammer ping` và health check OK); lỗi quyền dump PostgreSQL xử lý bằng `chmod 755`.
