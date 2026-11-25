---
title: Khắc phục thảm hoạ, clone và đổi tên Satellite
tags: [runbook, satellite, backup, capsule]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Administering Red Hat Satellite, chương 1, 4, 13 (PDF tr. 11–14, 22–32, 72–75); kinh nghiệm triển khai thực tế (ẩn danh)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Red Hat đưa ra bốn mô hình DR: snapshot VM, active/passive dùng chung storage, active/passive bằng backup/restore, và hai Satellite cùng active. Ba mô hình đầu giữ hostname, chỉ đổi DNS nên host không phải đăng ký lại; `satellite-clone` dựng bản sao trên máy mới, `satellite-change-hostname` đổi tên.

## Bốn mô hình DR

| Mô hình | Sao lưu thế nào | Khôi phục thế nào | Mất dữ liệu / nhược điểm |
|---|---|---|---|
| Snapshot VM | Hypervisor chụp snapshot định kỳ, sao lưu snapshot ra nơi khác | Khôi phục snapshot, đổi DNS nếu IP đổi | Mất thay đổi từ sau snapshot cuối |
| Active/passive, storage ngoài | `/var/lib/pulp` và `/var/lib/pgsql` trên NAS, nhân bản sang DC khác; máy passive là bản clone, tắt | Gắn storage vào máy passive, đổi DNS | Lệch theo chu kỳ nhân bản storage; cần CSDL ngoài |
| Active/passive, backup/restore | Backup offline định kỳ, restore sang máy passive rồi tắt nó | Bật máy passive, đổi DNS | Lệch theo tần suất backup/restore và thời gian restore |
| Hai Satellite cùng active | Mỗi DC một Satellite độc lập, host DC nào đăng ký vào Satellite DC đó | Đăng ký lại host sang Satellite còn sống | Phải chống lệch content (sync và CV giống nhau ở hai bên) |

Nguyên tắc chung:

- Hostname của Satellite **không đổi** khi khôi phục; IP được phép đổi, chỉ cần cập nhật bản ghi DNS → host vẫn kết nối, không phải đăng ký lại.
- Máy cũ phải tắt hẳn hoặc tách khỏi storage trước khi bật máy thay thế; hai bản clone không được chạy song song trong production.
- Diễn tập định kỳ trong môi trường cách ly: giả lập mất máy active (tắt máy, halt VM hoặc chặn bằng tường lửa), đổi DNS, rồi kiểm tra.

Kiểm tra sau mọi kịch bản:

```bash
hammer ping
satellite-maintain service status
satellite-maintain health check
```

## Từng mô hình

### Snapshot VM

1. Định lịch snapshot theo mức mất dữ liệu chấp nhận được và chính sách giữ snapshot.
2. Sao lưu snapshot ra ngoài hypervisor để chống hỏng chính hypervisor.
3. Dùng CSDL ngoài thì chụp CSDL cùng lịch; khi khôi phục, snapshot CSDL phải cùng lúc hoặc cũ hơn snapshot Satellite.
4. Khôi phục: chọn snapshot → khôi phục bằng công cụ hypervisor → cập nhật DNS nếu IP mới.

Snapshot chỉ nhất quán khi đã `satellite-maintain service stop` trước lúc chụp (xem [Sao lưu và khôi phục](backup-and-restore.md)). Trong triển khai thực tế, đội triển khai không được cấp quyền vCenter nên thay snapshot bằng bản backup offline; nên thống nhất quyền chụp snapshot với đội ảo hoá ngay từ thiết kế.

### Active/passive với storage ngoài

Yêu cầu Satellite đã dùng CSDL ngoài. Nhân bản `/var/lib/pulp`, `/var/lib/pgsql` lên storage dùng chung → clone máy active → tắt máy mới làm passive. Chọn gắn storage trên cả hai máy (luôn thấy dữ liệu mới) hoặc chỉ trên máy active (gắn lại khi chuyển đổi). Khi sự cố: xác nhận máy hỏng đã tắt hoặc tách storage (tránh ghi đồng thời) → đổi DNS → gắn storage nếu cần → kiểm tra.

### Active/passive với backup/restore

1. Lịch backup offline (full + incremental), mã hoá và đồng bộ thư mục backup tới nơi an toàn; mặc định Satellite lưu ở `/var/satellite-backup`.
2. Restore bản mới nhất lên máy passive; có thể tự động hoá việc restore định kỳ để rút ngắn thời gian chuyển đổi.
3. Tắt máy passive, đặt chính sách giữ bản sao lưu.
4. Khi sự cố: tắt máy active hỏng, ngừng đồng bộ backup sang passive → đổi DNS → kiểm tra.

### Hai Satellite cùng active

- Satellite thứ hai dựng bằng restore bản backup của Satellite đầu, nhưng **khác hostname và IP**.
- Chống lệch content: cả hai tự sync và tạo CV giống nhau (tự động hoá bằng `redhat.satellite.repository_sync`, `redhat.satellite.sync_plan`), hoặc một bên làm gốc rồi bên kia lấy qua Inter-Satellite Sync (có mạng) hay export/import (không có mạng).
- Tự động chạy `satellite-maintain health check` trên cả hai; khi một bên lỗi: xác nhận bên đó đã tắt, đăng ký lại toàn bộ host của DC đó sang Satellite còn lại (có thể dùng Ansible collection).

## Clone Satellite sang máy mới

Dùng để thử nâng cấp hoặc chuyển Satellite sang máy/HĐH khác. Công cụ `satellite-clone` **không** dùng cho Capsule: Capsule thì backup → restore trên máy mới → cấu hình lại.

Điều kiện máy đích: RHEL 9 cài tối thiểu, không cài thêm nhóm phần mềm hay ứng dụng bên thứ ba; có subscription Satellite; mạng cách ly để không nói chuyện với Satellite, Capsule, host thật; dung lượng ít nhất bằng máy nguồn. Cấu hình tuỳ biến nằm ngoài installer và backup phải tự sao lưu.

Trên máy nguồn, chạy bằng root:

```bash
du -sh /var/lib/pulp/
# Pulp dưới 500 GB: backup kèm Pulp
satellite-maintain backup offline --assumeyes /var/backup
# Pulp trên 500 GB (hoặc trên 100 GB với storage chậm như NFS): bỏ Pulp rồi chép riêng
satellite-maintain backup offline --skip-pulp-content --assumeyes /var/backup
satellite-maintain service stop
satellite-maintain service disable
rsync --archive --partial --progress --compress /var/lib/pulp/ <TARGET_FQDN>:/var/lib/pulp/
```

Nhánh dưới 500 GB cũng dừng và disable dịch vụ sau khi backup xong.

Trên máy đích, chạy bằng root:

```bash
# Đặt bản sao lưu vào /backup/ (chép hoặc mount), tắt máy nguồn rồi:
subscription-manager register
subscription-manager repos --disable=*
subscription-manager repos --enable=rhel-9-for-x86_64-appstream-rpms \
  --enable=rhel-9-for-x86_64-baseos-rpms \
  --enable=satellite-maintenance-6.19-for-rhel-9-x86_64-rpms
dnf install satellite-clone
# Bản sao lưu không nằm ở /backup/ thì sửa backup_dir trong /etc/satellite-clone/satellite-clone-vars.yml
satellite-clone
```

Sau khi clone:

1. Đăng nhập Web UI bằng `admin` với mật khẩu mặc định do công cụ đặt (`changeme`) và **đổi mật khẩu ngay**.
2. Cấu hình lại DHCP, DNS, TFTP, remote execution (clone tắt chúng để tránh đụng máy nguồn) và bật lại trên Web UI.
3. Content > Subscriptions > Manage Manifest > Refresh.
4. Làm theo `/usr/share/satellite-clone/logs/reassociate_capsules.txt` để gắn lại Capsule với lifecycle environment.
5. Cập nhật DNS cho IP mới; clone tự đặt hostname bằng hostname nguồn, muốn tên khác thì dùng `satellite-change-hostname` (mục dưới).
6. Có virt-who thì cài lại và chép `/etc/virt-who.d/` từ máy nguồn.
7. Loại bỏ máy nguồn.

## Đổi tên Satellite hoặc Capsule

Điều kiện: có backup trước khi đổi; `hostname` và `hostname -f` đều trả FQDN (nếu `hostname` trả tên ngắn thì `hostnamectl set-hostname <OLD_FQDN>` trước); có chứng chỉ mới nếu dùng chứng chỉ tuỳ chỉnh. Lệnh dừng toàn bộ dịch vụ và khởi động lại khi xong.

Satellite, chạy bằng root:

```bash
satellite-change-hostname <NEW_SATELLITE_FQDN> --username <USER> --password '<PASSWORD>'
# chứng chỉ tuỳ chỉnh: thêm --custom-cert "<CERT_PATH>" --custom-key "<KEY_PATH>"
```

Sau đó: đăng ký lại mọi host và Capsule; trên từng Capsule chạy

```bash
satellite-installer --foreman-proxy-foreman-base-url https://<NEW_SATELLITE_FQDN> \
  --foreman-proxy-trusted-hosts <NEW_SATELLITE_FQDN>
```

rồi `hammer capsule list` và `hammer capsule content synchronize --id <CAPSULE_ID>`; cập nhật cấu hình virt-who; cấu hình lại xác thực ngoài nếu có.

Capsule: trên Satellite tạo gói chứng chỉ mới cho tên mới, chép sang Capsule, chạy `satellite-change-hostname` trên Capsule:

```bash
capsule-certs-generate --certs-tar /root/<NEW_CAPSULE_FQDN>-certs.tar --foreman-proxy-fqdn <NEW_CAPSULE_FQDN>
satellite-change-hostname <NEW_CAPSULE_FQDN> --certs-tar /root/<NEW_CAPSULE_FQDN>-certs.tar \
  --username <USER> --password '<PASSWORD>'
```

Sau đó đăng ký lại host của Capsule đó, sửa Name/URL của Capsule ở Infrastructure > Capsules, thêm bản ghi DNS mới và xoá bản ghi cũ. Đường dẫn tới file `.tar` phải là đường dẫn đầy đủ.

Mật khẩu đưa trên dòng lệnh sẽ nằm trong lịch sử shell: xoá lịch sử hoặc dùng tài khoản tạm sau khi đổi tên.

## Lưu ý

- Mục tiêu RPO/RTO quyết định mô hình: snapshot và backup/restore rẻ nhưng mất dữ liệu từ lần sao lưu cuối; storage ngoài cần CSDL ngoài và storage nhân bản; hai active cần quy trình chống lệch content.
- Capsule đặt sau load balancer giảm phụ thuộc vào một Capsule, nhưng không thay được DR cho Satellite: khi Satellite hỏng, host chỉ còn tải gói từ Capsule (xem [Cứu host khi Satellite hỏng](client-rescue-when-satellite-down.md)).
- Diễn tập phải ghi lại thời gian thực tế của từng bước (restore, đổi DNS, sync Capsule) để biết RTO thật.

## Liên quan

- [Sao lưu và khôi phục](backup-and-restore.md)
- [Health check và xử lý sự cố](health-check-and-troubleshooting.md)
- [Cứu host khi Satellite hỏng](client-rescue-when-satellite-down.md)
- [Inter-Satellite Sync](../content/inter-satellite-sync.md)
- [Satellite Ansible Collection](../ansible/satellite-ansible-collection.md)
- [Chứng chỉ](../installation/certificates.md)
- [Đăng ký host](../hosts/host-registration.md)

## Nguồn

- Red Hat Satellite 6.19, Administering Red Hat Satellite, chương 1 Cloning Satellite Server (PDF tr. 11–14), chương 4 Preparing for disaster recovery and recovering from data loss (PDF tr. 22–32), chương 13 Renaming Satellite Server or Capsule Server (PDF tr. 72–75) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: không có quyền snapshot trên vCenter nên DR dựa vào backup offline; diễn tập restore trên VM cách ly cùng hostname đạt yêu cầu.
