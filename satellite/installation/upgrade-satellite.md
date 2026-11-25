---
title: Nâng cấp Satellite lên 6.19 và cập nhật bản vá 6.19.z
tags: [runbook, satellite, upgrade, capsule, backup]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Upgrading disconnected Red Hat Satellite to 6.19, chương 2–3 (PDF tr. 8–19); Administering Red Hat Satellite, phụ lục E (PDF tr. 132); tài liệu vận hành dự án (ẩn danh)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Lên 6.19 chỉ đi từ 6.18: sao lưu, chuẩn bị repo 6.19, chạy `satellite-maintain self-upgrade`, `upgrade check`, `upgrade run`, sync repo 6.19 rồi nâng cấp từng Capsule. Bản vá trong cùng phiên bản (6.19.z) dùng `satellite-maintain update check` và `update run`; gói OS dùng `satellite-maintain packages update`.

## Ba loại thay đổi

| Loại | Ví dụ | Lệnh |
|---|---|---|
| Upgrade (phiên bản) | 6.18.x → 6.19 | `satellite-maintain upgrade check` / `upgrade run` |
| Update (bản vá z-stream) | 6.19.x → 6.19.y | `satellite-maintain update check` / `update run` |
| Gói hệ điều hành nền | errata RHEL trên máy Satellite/Capsule | `satellite-maintain packages check-update` / `packages update` |

`dnf` trực tiếp bị khoá trên Satellite/Capsule và không được hỗ trợ; `satellite-maintain packages` tự chạy `satellite-installer` sau khi cài/cập nhật gói. Không dùng `satellite-maintain packages unlock` rồi `dnf update`; đã mở khoá thì khoá lại bằng `satellite-maintain packages lock` (Administering, phụ lục E).

## Lập kế hoạch (chương 2, PDF tr. 8–9)

- Lộ trình: chỉ từ 6.18 (mọi bản vá) lên 6.19. Bản thấp hơn phải lên tuần tự từng phiên bản theo tài liệu nâng cấp của từng bản.
- Mọi Satellite Server (ISS) cùng phiên bản. Capsule 6.18 và 6.17 vẫn chạy với Satellite 6.19 → nâng Capsule tách đợt được; Capsule sau load balancer nâng cuốn chiếu từng máy, không gián đoạn. Tính năng mới của 6.19 chỉ có trên Capsule đã nâng.
- Đọc release notes: API, Hammer, Satellite Ansible Collection có thể đổi; nội dung Flatpak không còn phục vụ host sau khi lên 6.19; module Puppet phải tương thích OpenVox 8 (Ruby 3.2).
- Thời gian dừng dịch vụ: Satellite khoảng 30 phút (rất lớn 1–2 giờ), mỗi Capsule khoảng 10 phút (rất lớn 15–30 phút). Có thể thử trước trên bản clone ([Sao chép và DR](../operations/disaster-recovery-and-cloning.md)).
- `satellite-installer` chạy trong quá trình nâng cấp và ghi đè file nó quản lý: kiểm tra sửa tay bằng `--noop`. Đã sửa tay DNS/DHCP thì trước đó chạy `satellite-installer --foreman-proxy-dhcp-managed false --foreman-proxy-dns-managed false`.
- Dùng Lightspeed trong Satellite: nạp container image 6.19 trước khi nâng cấp (`/media/sat6/setup_containers` từ ISO, hoặc `skopeo copy` ra `oci-archive` trên máy có mạng rồi nạp lại) — mục 3.1.

## Checklist trước khi chạy

```bash
satellite-maintain health check
hammer ping
rpm -q satellite
satellite-maintain upgrade list-versions
df -h /var /var/lib/pulp /var/lib/pgsql
```

- Không còn task đang chạy (Monitor → Satellite Tasks → Tasks); thông báo người dùng.
- Sao lưu: `satellite-maintain service stop` → snapshot VM (hoặc backup máy vật lý) → `satellite-maintain service start`; hoặc backup offline ([Sao lưu và khôi phục](../operations/backup-and-restore.md)). Đây cũng là đường rollback.
- Chạy trong `tmux`; mất kết nối thì xem `/var/log/foreman-installer/satellite.log`.
- Có biến `TMOUT` theo chính sách bảo mật làm bước kiểm tra lỗi: thêm `--whitelist="check-tmout-variable"` (ghi nhận trong tài liệu vận hành dự án).

## Nâng cấp Satellite không có Internet (mục 3.2, PDF tr. 12–15)

1. Sao lưu như trên.
2. Gỡ các file repo cũ khỏi `/etc/yum.repos.d/` (tài liệu hãng dùng `rm /etc/yum.repos.d/*`; nên chuyển sang thư mục sao lưu thay vì xoá).
3. Trên Satellite **có kết nối**: sync với download policy **Immediate** các repo `rhel-9-for-x86_64-baseos-rpms`, `rhel-9-for-x86_64-appstream-rpms`, `satellite-6.19-for-rhel-9-x86_64-rpms`, `satellite-maintenance-6.19-for-rhel-9-x86_64-rpms`, rồi xuất từng repo dạng syncable và chép sang Satellite cần nâng cấp:

```bash
hammer repository list --organization-label <ORG_LABEL>
hammer content-export complete repository --format syncable --id <REPO_ID>
```

4. Trên Satellite cần nâng cấp, tạo `/etc/yum.repos.d/upgrade.repo` trỏ `baseurl=file:///<EXPORT_DIR>/...` vào bản xuất: BaseOS `content/dist/rhel9/9/x86_64/baseos/os`, AppStream `content/dist/rhel9/9/x86_64/appstream/os`, Satellite `content/dist/layered/rhel9/x86_64/satellite/6.19/os/`, Maintenance `content/dist/layered/rhel9/x86_64/sat-maintenance/6.19/os/`; mỗi mục `enabled=1`, `gpgcheck=0`, `metadata_expire=-1`, riêng `[satellite-6.19-for-rhel-9-x86_64-rpms]` để `enabled=0`.
5. Nâng `satellite-maintain` và kiểm tra điều kiện:

```bash
satellite-maintain self-upgrade --maintenance-repo-label=satellite-maintenance-6.19-for-rhel-9-x86_64-rpms
satellite-maintain upgrade check --whitelist="repositories-validate,repositories-setup"
```

   Lần đầu `upgrade check` có thể hỏi tài khoản admin của hammer và ghi vào `/etc/foreman-maintain/foreman-maintain-hammer.yml`. Xử lý hết lỗi nó báo.
6. Sửa `upgrade.repo`: bật `satellite-6.19-for-rhel-9-x86_64-rpms` (`enabled=1`).
7. Nâng cấp:

```bash
satellite-maintain upgrade run --whitelist="repositories-validate,repositories-setup"
```

   Thiếu hoặc lệch gói: tải gói lẻ và cài cục bộ như khi cài disconnected ([Cài Satellite disconnected](install-satellite-disconnected.md), Bước 3).
8. `satellite-maintain` yêu cầu thì `reboot`.
9. Việc sau nâng cấp (mục 3.2, bước 10–12): plugin OpenSCAP thiếu nội dung mặc định thì `foreman-rake foreman_openscap:bulk_upload:default`; Configure → Discovery Rules gán lại Organization/Location; Lightspeed trong Satellite: chép `cvemap.xml` mới vào `/var/lib/foreman/`; so template mặc định đã thay đổi với các template đã clone; Hosts → Discovered hosts (Any Organization) reboot mọi discovered host.

Satellite **có kết nối** (theo tài liệu vận hành dự án, dựa trên bản Upgrading connected — guide này chưa có trong bộ PDF đã đối chiếu): `subscription-manager repos --enable satellite-maintenance-6.19-for-rhel-9-x86_64-rpms`, rồi `satellite-maintain self-upgrade`, `satellite-maintain upgrade check`, `satellite-maintain upgrade run` không cần `--whitelist`.

## Sync repo 6.19 (mục 3.3)

Trước khi nâng Capsule và client: Content → Red Hat Repositories bật và sync **Red Hat Satellite Capsule 6.19**, **Satellite Maintenance 6.19**, **RHEL 9 BaseOS/AppStream**, **Red Hat Satellite Client 6** cho từng major RHEL của host (và **Client 6 OpenVox** nếu dùng Puppet). Dùng Puppet: thêm repo OpenVox vào CV của host, publish/promote. Sync lỗi: Content → Subscriptions → Manage Manifest → **Refresh**; không bao giờ xoá manifest (mất entitlement của mọi host).

## Nâng cấp Capsule (mục 3.4)

Điều kiện: OS Capsule đăng ký vào Satellite đã nâng cấp; nếu Capsule gắn với một Content View thì thêm repo Capsule 6.19 và Maintenance 6.19 vào CV đó, publish và promote tới LE của Capsule. Trên Capsule:

```bash
subscription-manager repos --enable satellite-maintenance-6.19-for-rhel-9-x86_64-rpms
satellite-maintain self-upgrade
satellite-maintain upgrade check
satellite-maintain upgrade run
```

- Sao lưu trước (snapshot hoặc backup đầy đủ); reboot nếu được yêu cầu.
- Có thể nâng từ xa bằng job template **Capsule Upgrade Playbook** (REX).
- Tarball chứng chỉ của Capsule (`/root/<CAPSULE_FQDN>-certs.tar`) phải còn nguyên.
- Capsule sau LB: đặt member về Disabled/Offline trên LB, nâng, kiểm tra, đưa vào lại, rồi mới sang máy tiếp theo ([Capsule sau load balancer](capsule-load-balancer.md)).
- Sau đó trên Satellite: Infrastructure → Capsules → **Refresh features**, phiên bản hiển thị 6.19; Optimized Sync nếu cần.

## Sau nâng cấp

- DB ngoài trên RHEL 8: có thể chuyển sang máy RHEL 9 trong đợt này (sao lưu, khôi phục sang máy mới, đổi `--foreman-db-host`, `--katello-candlepin-db-host`, `--foreman-proxy-content-pulpcore-postgresql-host` nếu đổi tên) — mục 3.5.
- Lightspeed trong Satellite: trên host chạy `insights-client --register --force` (có thể qua REX) — mục 3.6.
- Puppet: nâng `puppet-agent` 7 lên OpenVox agent 8 bằng `dnf upgrade puppet-agent` trên host (cần repo Client 6 OpenVox) — mục 3.7.
- Kiểm tra và mở chuỗi sao lưu mới:

```bash
rpm -q satellite
satellite-maintain service status
hammer ping
satellite-maintain health check
hammer capsule list
```

Administer → About hiển thị 6.19 và trạng thái các Capsule.

## Cập nhật bản vá 6.19.z (update)

Phần này lấy từ tài liệu vận hành dự án (dựa trên guide Updating Red Hat Satellite, chưa có trong bộ PDF đã đối chiếu số trang):

1. Sao lưu (snapshot hoặc backup offline), kiểm tra không còn task chạy.
2. Satellite có kết nối: bật `satellite-maintenance-6.19-for-rhel-9-x86_64-rpms`, rồi:

```bash
satellite-maintain update check
satellite-maintain update run
```

   Lần đầu `update check` hỏi tài khoản admin của hammer. Reboot nếu được yêu cầu.
3. Satellite không có Internet: làm mới nguồn gói cục bộ bằng nội dung 6.19.z mới (xuất syncable từ Satellite có kết nối như phần nâng cấp, hoặc ISO Satellite mới nhất), rồi `update check` / `update run`. Có cần `--whitelist="repositories-validate,repositories-setup"` như khi upgrade hay không: **chưa kiểm chứng**, đối chiếu guide Updating trước khi làm.
4. Capsule sau Satellite: publish/promote phiên bản mới của CV dành cho Capsule để Capsule thấy gói mới, rồi trên từng Capsule `satellite-maintain update check` và `satellite-maintain update run`. Phiên bản Capsule phải bằng Satellite.
5. Kiểm tra như phần sau nâng cấp; Capsule chạy Optimized Sync.

Gói OS nền (không đổi phiên bản Satellite):

```bash
satellite-maintain packages check-update
satellite-maintain packages update
satellite-maintain service status
dnf needs-restarting -r
```

## Lỗi hay gặp

| Hiện tượng | Nguyên nhân | Xử lý |
|---|---|---|
| `upgrade check` báo lỗi kiểm tra repository trên máy không Internet | Thiếu `--whitelist="repositories-validate,repositories-setup"` | Chạy lại với whitelist như mục 3.2 |
| `upgrade run` dừng vì thiếu gói | Bản xuất thiếu gói phụ thuộc | Tải gói lẻ, cài cục bộ, chạy lại |
| Sửa tay DNS/DHCP bị mất | Installer ghi đè file quản lý | Tắt `dns-managed`/`dhcp-managed` trước; khôi phục bằng `puppet filebucket` |
| Sync repo 6.19 lỗi | Manifest cũ | Refresh manifest, không xoá |
| Capsule không nâng được | Chưa sync repo 6.19 hoặc CV của Capsule chưa có repo 6.19 | Mục "Sync repo 6.19" và điều kiện nâng Capsule |

## Liên quan

- [Cài Satellite disconnected](install-satellite-disconnected.md) · [Cài Capsule Server](install-capsule.md) · [Capsule sau load balancer](capsule-load-balancer.md) · [Chứng chỉ SSL](certificates.md)
- [Sao lưu và khôi phục](../operations/backup-and-restore.md) · [Health check](../operations/health-check-and-troubleshooting.md) · [Inter-Satellite Sync](../content/inter-satellite-sync.md) · [Nâng cấp RHEL bằng Leapp](../hosts/rhel-major-upgrade-leapp.md)

## Nguồn

- Red Hat Satellite 6.19, Upgrading disconnected Red Hat Satellite to 6.19, chương 2 Preparing for Satellite upgrade (PDF tr. 8–9), chương 3 Upgrading Red Hat Satellite, mục 3.1–3.7 (PDF tr. 10–19) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Administering Red Hat Satellite, phụ lục E Managing packages on the base operating system (PDF tr. 132).
- Red Hat Satellite 6.19, Updating Red Hat Satellite (chưa có trong bộ PDF đã trích; nội dung update 6.19.z ở trang này tóm từ tài liệu vận hành dự án, cần đối chiếu).
- Tài liệu vận hành dự án (ẩn danh), 2026: checklist trước cập nhật (task, tmux, `--noop`, DNS/DHCP), `update check/run`, `packages update`, whitelist `check-tmout-variable`, cập nhật Capsule qua CV dành cho Capsule.
