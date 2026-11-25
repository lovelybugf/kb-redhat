---
title: Sizing phần cứng, dung lượng và tuning hiệu năng Satellite
tags: [reference, satellite, performance, capsule, storage]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Tuning performance of Red Hat Satellite, chương 2–5 (PDF tr. 8–22); Installing Satellite Server in a connected network environment, chương 1 và 4 (PDF tr. 8–13, 24–25); Installing Capsule Server, mục 1.2–1.4 (PDF tr. 7–9); Administering Red Hat Satellite, chương 3 (PDF tr. 17–18)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Tối thiểu Satellite cần 4 core, 20 GB RAM, 4 GB swap và khoảng 300 GB cho `/var/lib/pulp`; Capsule cần 4 core, 12 GB RAM. Khi số host tăng, áp `satellite-installer --tuning <size>` (default tới 5.000 host), rồi mới tinh chỉnh Puma, Dynflow, PostgreSQL, httpd qua `custom-hiera.yaml`.

## Yêu cầu tối thiểu

| | Satellite Server | Capsule Server |
|---|---|---|
| Hệ điều hành | RHEL 9 x86_64, bản minor mới nhất lúc cài (không EUS), `@Base` | Như Satellite |
| CPU | 4 core | 4 core |
| RAM | 20 GB | 12 GB |
| Swap | 4 GB | 4 GB |
| Khác | Máy mới, chỉ chạy Satellite; FQDN chữ thường; SELinux enforcing hoặc permissive; đồng bộ thời gian; locale UTF-8 | Cùng phiên bản với Satellite |

Chạy được trên máy vật lý hoặc VM trên hypervisor được chứng nhận cho RHEL.

## Dung lượng thư mục

Satellite Server:

| Thư mục | Lúc cài | Khi vận hành |
|---|---|---|
| `/var/log` | 10 MB | 10 GB |
| `/var/lib/pgsql` | 1 GB | 20 GB |
| `/usr` | 10 GB | — |
| `/opt/puppetlabs` | 500 MB | — |
| `/var/lib/pulp` | 1 MB | 300 GB |
| `/var/lib/containers` (khi dùng Lightspeed trong Satellite) | 20 GB | 30 GB |

Capsule Server:

| Thư mục | Lúc cài | Khi vận hành |
|---|---|---|
| `/var/lib/pulp` | 1 MB | 300 GB |
| `/var/lib/pgsql` | 100 MB | 20 GB |
| `/usr` | 3 GB | — |
| `/opt/puppetlabs` | 500 MB | — |

- Số "khi vận hành" đo với repo RHEL 7, 8, 9 đã sync; thực tế phụ thuộc số repo, download policy, số phiên bản CV giữ lại, số LE trên Capsule.
- CSDL của Capsule tăng theo số LE, CV và repo được sync; ở môi trường rất lớn `/var/lib/pgsql` trên Capsule có thể gấp 2–3 lần trên Satellite.
- Đặt `/var` trên LVM để mở rộng dễ ([LVM](https://github.com/lovelybugf/kb-base-devops/blob/main/linux/storage-lvm.md)); dùng lưu trữ độ trễ thấp (SSD) cho `/var/lib/pulp` và `/var/lib/pgsql`; không dùng GFS2.
- `/tmp` tách riêng phải mount `exec` (puppetserver cần). Không dùng symbolic link cho `/var/lib/pulp`.
- `/var/lib/pulp` trên NFS phải khai báo SELinux context trong `/etc/fstab`, rồi `restorecon`:

```text
<NFS_SERVER>:/<EXPORT>  /var/lib/pulp  nfs  context="system_u:object_r:pulpcore_var_lib_t:s0"  1 2
```

```bash
restorecon -R /var/lib/pulp
```

## Sizing tham khảo từ triển khai thực tế (ẩn danh)

Quy mô dưới vài nghìn host, RHEL 8/9/10, khoảng 11–15 repo Red Hat:

| Máy | vCPU / RAM | Ổ dữ liệu | Ghi chú |
|---|---|---|---|
| Satellite | 8 / 32 GB | 700 GB – 1 TB: `/var/lib/pulp` 500–650 GB, `/var/lib/pgsql` 50–100 GB, `/var/log` 30–40 GB, phần còn lại dự phòng trong VG | Profile default; 8/32 vừa đủ mức medium nếu quy mô tăng |
| Capsule | 4 / 16 GB | 500 GB – 1 TB: `/var/lib/pulp` 300–700 GB, `/var/lib/pgsql` 50 GB, `/var/log` 30 GB | Capsule giữ nhiều LE cần nhiều dung lượng hơn |
| Đích sao lưu | — | Ổ riêng hoặc NFS | Bản full không nén Pulp, cần thêm khoảng 20% so với `/var/lib/pulp` |

- Đầu vào để ước lượng: số host trong 3 năm, số repo, download policy, chu kỳ publish và số phiên bản CV giữ lại, số LE mỗi Capsule, thời gian lưu báo cáo ARF, số host đăng ký đồng thời, yêu cầu N+1 cho Capsule.
- Cảnh báo dung lượng ở 80% cho `/var/lib/pulp`, `/var/lib/pgsql`, `/var/log` và thư mục sao lưu ([quản lý dung lượng](../operations/storage-management.md)).
- Lưu trữ all-flash, thử bằng `fio` trước khi cài; mục tiêu thiết kế đã dùng khoảng 1.500 IOPS cho `/var/lib/pulp` và `/var/lib/pgsql` (con số tự đặt, không phải yêu cầu của hãng).
- Lab: VM Satellite trên ESXi bị balloon bộ nhớ làm Pulp hết RAM (OOM) — đặt reservation 100% RAM cho VM Satellite/Capsule.

## Tuning profile (`--tuning`)

| Profile | Số host | RAM | CPU core |
|---|---|---|---|
| `default` | 0 – 5.000 | 20 GB | 4 |
| `medium` | 5.001 – 10.000 | 32 GB | 8 |
| `large` | 10.001 – 20.000 | 64 GB | 16 |
| `extra-large` | 20.001 – 60.000 | 128 GB | 32 |
| `extra-extra-large` | trên 60.000 | 256 GB trở lên | 48 trở lên |

Thứ tự ưu tiên khi áp: `/etc/foreman-installer/custom-hiera.yaml` > profile đã chọn (`/usr/share/foreman-installer/config/foreman.hiera/tuning/sizes/`) > profile nền (`.../tuning/common.yaml`).

Chạy bằng root trên Satellite:

```bash
cp /etc/foreman-installer/custom-hiera.yaml /etc/foreman-installer/custom-hiera.original.yaml
ls /usr/share/foreman-installer/config/foreman.hiera/tuning/sizes/
# So sánh custom-hiera.yaml với common.yaml và file profile; xoá khoá trùng để profile có hiệu lực
satellite-installer --tuning medium
satellite-maintain health check
```

- Tuning profile **không** dùng cho Capsule.
- Khoá nào đã đặt trong `custom-hiera.yaml` sẽ đè lên profile — đổi profile sau đó không có tác dụng với khoá đó.

## Các điểm tinh chỉnh chính

Thử trên môi trường không phải production, có bản sao lưu, có giám sát trước và sau; hầu hết thay đổi cần restart Satellite.

**Phần cứng và OS**
- Nhiều core giúp PostgreSQL, Puppet; nhiều RAM giúp PostgreSQL cache; SSD cho CSDL; mạng Satellite ↔ Capsule ít jitter, độ trễ thấp; BIOS đặt chế độ performance thay vì tiết kiệm điện.
- Bật `tuned` (chạy bằng root): `satellite-maintain packages install tuned`, `systemctl enable --now tuned`, `tuned-adm list`, `tuned-adm profile <PROFILE>`. Tài liệu 6.19 không chỉ định profile; thường chọn `virtual-guest` cho VM, `throughput-performance` cho máy vật lý.
- Tắt Transparent Hugepage vì PostgreSQL và Redis chạy kém khi bật (theo KB của Red Hat về THP); lợi ích giảm nếu CSDL ở máy riêng.

**Puma (Web UI, API, đăng ký)**
- Mặc định installer tự tính số worker: `min(CPU_COUNT * 1.5, RAM_IN_GB - 1.5)`; mỗi worker tốn khoảng 1 GiB RAM; mặc định 5 thread.
- Gợi ý của Red Hat (đo bằng đăng ký đồng thời): 16 thread; worker 4–6 (default), 8–12 (medium), 12–18 (large), 16–24 (extra-large), 20–26 (extra-extra-large). Số worker ảnh hưởng nhiều hơn số thread.
- Satellite thiên về content (publish/promote nhiều) có thể giới hạn Puma để dành tài nguyên cho Pulp và PostgreSQL.

```bash
cat /etc/systemd/system/foreman.service.d/installer.conf   # giá trị hiện tại
satellite-installer \
  --foreman-foreman-service-puma-workers 4 \
  --foreman-foreman-service-puma-threads-max 16
```

**Dynflow (tác vụ nền)**: nhiều publish/promote, sync, sync Capsule đồng thời → tăng số worker Sidekiq (hiệu quả hơn tăng concurrency quá 5):

```bash
satellite-installer --foreman-dynflow-worker-instances 3
systemctl -a | grep 'dynflow-sidekiq@worker-[0-9]'
```

**Apache httpd**: hết process thì trả HTTP 503 và kéo theo lỗi dây chuyền. Giá trị dưới đây tương đương `--tuning medium`, chỉ đặt khi có nhu cầu riêng (thêm vào `custom-hiera.yaml` rồi chạy `satellite-installer`):

```yaml
apache::mod::event::serverlimit: 64
apache::mod::event::maxrequestworkers: 1024
apache::mod::event::maxrequestsperchild: 4000
```

**PostgreSQL**: `shared_buffers` từ 2 GiB tới tối đa 25% RAM; `work_mem` 8 MB là đủ; `autovacuum_vacuum_cost_limit` 2000 như các profile từ medium trở lên.

```yaml
postgresql::server::config_entries:
  max_connections: 1000
  shared_buffers: 2GB
  work_mem: 8MB
  autovacuum_vacuum_cost_limit: 2000
```

Đo CSDL bằng `pgbench` (gói `postgresql-contrib`) chỉ trên hệ thống thử nghiệm có bản sao lưu; bảng lớn nhất xem bằng script `postgres-size-report` trong repo satellite-support.

**REX pull (MQTT)**: Mosquitto trên Capsule mặc định 1024 host. Nâng giới hạn trong `custom-hiera.yaml` trên Capsule rồi chạy `satellite-installer`:

```yaml
systemd::dropin_files:
  limits.conf:
    unit: mosquitto.service
    content: "[Service]\nLimitNOFILE=5000\n"
```

Job đồng thời lớn làm cạn kết nối Capsule: đặt rate limit, resend interval, TTL trên Satellite:

```bash
satellite-installer \
  --foreman-proxy-plugin-remote-execution-script-mqtt-rate-limit <RATE_LIMIT> \
  --foreman-proxy-plugin-remote-execution-script-mqtt-resend-interval <SECONDS> \
  --foreman-proxy-plugin-remote-execution-script-mqtt-ttl <SECONDS>
```

**Capsule**: đăng ký đồng thời rất cao gây timeout Capsule → Satellite; tăng `apache::mod::proxy::proxy_timeout: 600` trong `custom-hiera.yaml` của Capsule.

## Capsule: số liệu thử nghiệm của Red Hat

| Cấu hình | CPU | RAM |
|---|---|---|
| minimal | 4 | 12 GB |
| large | 8 | 24 GB |
| extra large | 16 | 46 GB |

- Tải content đồng thời: gấp đôi tài nguyên Capsule thì thời gian tải trung bình giảm khoảng 40–50%. Với 8 CPU/24 GB, Satellite chỉ phục vụ được khoảng 500 client tải đồng thời, Capsule cùng cấu hình phục vụ hơn 1.000.
- Đăng ký đồng thời: nút thắt là tốc độ CPU; tăng tài nguyên Capsule chỉ cải thiện tối đa khoảng 9%. Capsule không giảm tải cho thao tác dùng nhiều API Satellite như đăng ký hay cập nhật package profile.
- REX (SSH và Ansible) trên 500, 2.000, 4.000 host: mọi cấu hình chạy được, trừ minimal không hoàn tất 4.000 host.

## PostgreSQL ngoài

- Cân nhắc khi: REX thường xuyên, sync/publish dày (I/O cao), rất nhiều host hoặc nhiều content. Lợi: giải phóng RAM/CPU trên Satellite, tinh chỉnh CSDL độc lập. Hại: thêm một máy phải vá và bảo trì; Satellite dừng khi máy CSDL lỗi; độ trễ mạng làm chậm.
- Satellite 6.19 hỗ trợ PostgreSQL 13; Red Hat không hỗ trợ sao lưu, nâng cấp, tuning CSDL ngoài. Query chậm hơn 1 giây thường do quy mô lớn, chuyển CSDL ra ngoài chưa chắc giải quyết — mở case với Red Hat.

## Lưu ý từ thực tế

- Lab: publish một CV có filter mất khoảng 24 phút, cùng CV không filter khoảng 30 giây — tính thời gian publish vào cửa sổ bảo trì khi dùng filter.
- Mọi thay đổi tuning đều chạy lại `satellite-installer`; kiểm tra `satellite-maintain health check` và `hammer ping` ngay sau đó.

## Liên quan

- [Thành phần và dịch vụ](components-and-services.md) · [Lập kế hoạch triển khai](deployment-planning.md) · [Cổng mạng](network-ports.md)
- [Quản lý dung lượng](../operations/storage-management.md) · [Giám sát và metrics](../operations/monitoring-and-metrics.md)
- [Download policy](../content/download-policy.md) · [Remote execution](../hosts/remote-execution.md)

## Nguồn

- Red Hat Satellite 6.19, Tuning performance of Red Hat Satellite, chương 2 Selecting an installation size (PDF tr. 8), chương 4 Hardware and operating system configuration (PDF tr. 10–11), chương 5 Configuring Satellite for performance: Puma, httpd, Dynflow, pull REX, PostgreSQL, Redis, Capsule (PDF tr. 12–22) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Installing Satellite Server in a connected network environment, chương 1 Planning: 1.1–1.4 (PDF tr. 8–11); chương 4 Tuning with predefined profiles (PDF tr. 24–25) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Installing Capsule Server, mục 1.2–1.4 System and storage requirements (PDF tr. 7–9) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Administering Red Hat Satellite, chương 3 Migrating to external databases, mục 3.2 (PDF tr. 17–18); Overview, concepts, and deployment considerations, mục 11.1.1 (PDF tr. 42) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: cấu hình 8 vCPU/32 GB cho Satellite và 4 vCPU/16 GB cho Capsule đã chạy production và qua UAT; bố cục ổ đĩa; OOM do balloon bộ nhớ và thời gian publish có filter đo trên lab.
