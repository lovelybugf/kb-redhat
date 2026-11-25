---
title: Nâng cấp RHEL lên major tiếp theo bằng Leapp qua Satellite (kèm nâng minor)
tags: [runbook, satellite, rhel, upgrade, remote-execution, content-view]
product: Red Hat Satellite 6.19, RHEL 7–10
source: "Red Hat Satellite 6.19 — Managing hosts, chương 11, 12 (PDF tr. 73–79); tài liệu vận hành dự án (ẩn danh), 2026"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Satellite nâng cấp tại chỗ RHEL 7→8, 8→9, 9→10 bằng hai job REX: Preupgrade check with Leapp (xem báo cáo, xử lý mọi Inhibitor) rồi Upgrade with Leapp. Điều kiện then chốt: repo BaseOS/AppStream của major đích đã sync và nằm trong Content View mà host đang dùng, host đã lên minor mới nhất của bản nguồn, có snapshot để rollback.

## Phạm vi

| Lộ trình | Chuẩn bị host theo tài liệu RHEL |
|---|---|
| RHEL 7 → 8 | *Upgrading from RHEL 7 to RHEL 8* (nguồn phải là 7.9) |
| RHEL 8 → 9 | *Upgrading from RHEL 8 to RHEL 9* |
| RHEL 9 → 10 | *Upgrading from RHEL 9 to RHEL 10* |

- Satellite 6.19 cho chọn **phiên bản minor đích** ở cả bước pre-upgrade lẫn upgrade (job category *Leapp - Preupgrade* / *Leapp - Upgrade*); hai mục nhanh trong menu Schedule a Job thì lấy bản mới nhất.
- Không dùng cho chính Satellite/Capsule: Satellite 6.19 chỉ chạy trên RHEL 9 và vá OS bằng `satellite-maintain` — xem [Nâng cấp Satellite](../installation/upgrade-satellite.md).
- Máy CentOS 7, Oracle Linux 7/8 muốn thành RHEL: dùng Convert2RHEL (Managing hosts chương 12, PDF tr. 76–79), không phải Leapp.

## Checklist trước khi làm

- [ ] Đọc mục *Planning an upgrade* của lộ trình tương ứng: phần cứng, driver bị bỏ, thay đổi bảo mật, ứng dụng cần actor riêng.
- [ ] Host đăng ký Satellite, Simple Content Access (`subscription-manager status` báo *Content Access Mode is set to Simple Content Access*).
- [ ] REX chạy được tới host (khoá SSH của Capsule đã phân phối) — thử một job đơn giản trước.
- [ ] Nội dung major đích đã sẵn trong CV/LE của host và trên Capsule phục vụ host (Bước 1).
- [ ] Backup: VM snapshot, ReaR hoặc LVM snapshot; có quyền vào console (quá trình nâng cấp reboot vào môi trường riêng, mất từ vài phút tới vài giờ).
- [ ] Khung giờ bảo trì, đầu mối ứng dụng kiểm thử sau nâng cấp.
- [ ] Gỡ mount (và comment trong `/etc/fstab`) các filesystem dữ liệu không cần cho hệ điều hành.
- [ ] Host đang quản lý cấu hình bằng Puppet/OpenVox và lên RHEL 10: kiểm tra agent đã có cho RHEL 10 chưa.

## Bước 1 — Chuẩn bị nội dung trên Satellite

1. **Content → Red Hat Repositories**: bật và sync BaseOS + AppStream của major đích (ví dụ `rhel-9-for-x86_64-baseos-rpms`, `rhel-9-for-x86_64-appstream-rpms`) và **Red Hat Satellite Client 6** cho major đích (`satellite-client-6-for-rhel-9-x86_64-rpms`). Tài liệu nâng cấp RHEL yêu cầu repo đích khớp bản minor đích; đối chiếu trước khi chọn bản rolling hay bản theo minor.

```bash
hammer repository-set enable \
  --organization "<ORG>" \
  --product "Red Hat Enterprise Linux for x86_64" \
  --name "Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)" \
  --releasever "<TARGET_RELEASEVER>" \
  --basearch "x86_64"
```

   Làm tương tự cho AppStream và Satellite Client; sau đó sync (Content → Sync Status hoặc `hammer repository synchronize`).

2. Thêm các repo đích vào **Content View host đang dùng** (giữ nguyên repo bản nguồn), hoặc tạo Content View/Composite riêng cho đợt nâng cấp chứa cả repo nguồn lẫn đích. Publish, promote tới LE của host.

```bash
hammer content-view add-repository --organization "<ORG>" --name "<CONTENT_VIEW>" --repository-id <REPO_ID>
hammer content-view publish --organization "<ORG>" --name "<CONTENT_VIEW>" --description "Them repo major dich cho Leapp"
hammer content-view version promote --organization "<ORG>" --content-view "<CONTENT_VIEW>" --version <VERSION> --to-lifecycle-environment "<LIFECYCLE_ENV>"
```

3. Chờ Capsule phục vụ host sync xong LE đó (Infrastructure → Capsules → Capsule → Content).
4. Nếu chuyển host sang CV mới: sửa CV/LE trên chính host (sửa Activation Key không tác động host đã đăng ký — xem [Activation Key chỉ áp dụng lúc đăng ký](activation-key-registration-only.md)).

## Bước 2 — Chuẩn bị host

Chạy bằng root trên host (hoặc gửi bằng job REX *Run Command*):

```bash
subscription-manager status
subscription-manager release --unset
dnf update -y
reboot
```

Sau reboot (host phải đang chạy kernel mới nhất đã cài):

```bash
dnf install -y leapp-upgrade
subscription-manager repos --list-enabled
df -h /var
```

RHEL 7 dùng `yum` thay `dnf` và cần bật thêm `rhel-7-server-extras-rpms`.

## Bước 3 — Pre-upgrade check qua Satellite

1. (Tuỳ chọn) Muốn sửa template: clone *Run preupgrade via Leapp* / *Run upgrade via Leapp*, rồi **Administer → Remote Execution Features** → `leapp_preupgrade` / `leapp_upgrade` → chọn template mới → **Submit**.
2. **Hosts → All Hosts** → chọn host:
   - lên bản mới nhất: **Schedule a Job → Preupgrade check with Leapp**;
   - lên bản minor cụ thể: **Schedule a job** → Job category **Leapp - Preupgrade** → chọn phiên bản đích trong wizard.
3. Xem kết quả (chỉ có ở giao diện cũ): **Monitor → Jobs** → job → menu **Rerun all → Legacy UI** → tab **Leapp preupgrade report**.
   - **Inhibitor**: chắc chắn làm hỏng nâng cấp, bắt buộc xử lý.
   - **Has Remediation**: có lệnh sửa (tick rồi **Fix Selected** để Satellite chạy bằng REX) hoặc chỉ có gợi ý (Hint) để sửa tay.
4. Sửa xong → **Rerun** → **Submit**, lặp tới khi không còn Inhibitor.

Trên host, báo cáo chi tiết nằm ở `/var/log/leapp/leapp-report.txt` (bản JSON `leapp-report.json`), log ở `/var/log/leapp/`. Chạy tay trên host khi cần thử nhanh: `leapp preupgrade --target <TARGET_VERSION>`.

## Bước 4 — Xử lý Inhibitor thường gặp

Bảng dưới là các nhóm lỗi điển hình theo tài liệu nâng cấp RHEL; luôn làm theo đúng nội dung báo cáo của host.

| Báo cáo nói gì | Hướng xử lý |
|---|---|
| Thiếu câu trả lời trong answer file | Đọc `/var/log/leapp/answerfile`, trả lời bằng `leapp answer --section <SECTION>.<KEY>=<VALUE>` |
| Không tìm thấy repo của hệ đích | Quay lại Bước 1: repo đích chưa có trong CV version của host, chưa promote, hoặc Capsule chưa sync |
| Kernel mới nhất chưa được dùng | Reboot vào kernel mới nhất |
| Driver kernel đã bị bỏ ở bản đích đang được nạp | Gỡ module (`modprobe -r <MODULE>`) và chặn nạp lại nếu không cần; nếu cần thì hỏi hãng phần cứng |
| Không đủ dung lượng | Mở rộng filesystem chứa `/var` — [mở rộng LV](https://github.com/lovelybugf/kb-base-devops/blob/main/linux/storage-lvm.md) |
| RHEL 8 → 9: đăng nhập root qua SSH có thể hỏng | Ghi rõ `PermitRootLogin` trong `/etc/ssh/sshd_config` theo chính sách |
| RHEL 8 → 9: firewalld `AllowZoneDrifting` | Đặt `AllowZoneDrifting=no` trong `/etc/firewalld/firewalld.conf` |
| RHEL 9 → 10: cấu hình mạng kiểu ifcfg | Chuyển sang keyfile của NetworkManager nếu báo cáo yêu cầu (`nmcli connection migrate`) |

## Bước 5 — Nâng cấp

1. **Hosts → All Hosts** → chọn host → **Schedule a Job → Upgrade with Leapp** (bản mới nhất), hoặc **Schedule a job** → **Leapp - Upgrade** để chọn phiên bản đích.
2. Trong wizard chọn có tự reboot sau khi chuẩn bị xong hay không; với nhiều host, đặt Concurrency level để không dồn tải lên Capsule.
3. Theo dõi job ở **Monitor → Jobs** và console của máy: host reboot vào môi trường nâng cấp, cài gói bản mới, rồi reboot lần nữa. Trong lúc nâng cấp Leapp để SELinux ở chế độ permissive.

## Bước 6 — Sau nâng cấp

Chạy bằng root trên host:

```bash
cat /etc/redhat-release
uname -r
subscription-manager release --show
subscription-manager repos --list-enabled
dnf repolist
getenforce
subscription-manager facts --update
```

Tìm gói còn lại của bản nguồn (ví dụ nâng 8 → 9 thì tìm `el8`, kèm `el7` nếu máy từng nâng từ 7):

```bash
rpm -qa | grep -e '\.el8' | grep -v -e gpg-pubkey -e leapp | sort
```

- Đưa SELinux về `enforcing` nếu trước đó dùng enforcing; dọn gói còn lại của bản cũ, gói và dữ liệu leapp theo mục *Performing post-upgrade tasks* của tài liệu RHEL.
- Trên Satellite: trang host phải hiện OS mới sau khi facts cập nhật; chuyển host sang Content View/host group dành cho major mới, gỡ repo bản cũ khỏi CV khi không còn host nào cần; cột **RHEL Lifecycle status** (Hosts → All Hosts → Manage columns → Content) giúp theo dõi host còn ở bản sắp hết hỗ trợ.
- Kiểm thử ứng dụng, sau đó mới xoá snapshot.

## Rollback

Leapp không có thao tác hoàn tác tự động: rollback = khôi phục VM snapshot, bản ReaR hoặc LVM snapshot đã chuẩn bị ở checklist. Giữ snapshot tới khi ứng dụng chạy ổn định; khôi phục xong thì chạy `subscription-manager facts --update` để Satellite nhận lại OS cũ.

## Nâng minor trong cùng major (ví dụ 9.x lên 9.y)

Không cần Leapp: host chỉ thấy gói có trong CV version đang ở LE của nó, nên đưa gói bản minor mới vào CV rồi `dnf update`. Trước hết xác định host thuộc trường hợp nào:

| Kiểm tra | Ở đâu | Ý nghĩa |
|---|---|---|
| Repo trong CV là bản rolling hay bản minor | Content → Content Views → CV → Repositories | Tên có `9` = rolling; có `9.6` = khoá minor/EUS |
| CV version ở LE của host publish khi nào | Tab Versions | Publish trước ngày bản minor mới phát hành thì chưa có gói mới |
| Library đã sync sau ngày phát hành chưa | Content → Sync Status | Sync plan phải chạy sau ngày GA |
| Host có bị khoá release | Host: `subscription-manager release --show`; Satellite: trang host → Content | Đang đặt release = minor cũ thì phải bỏ hoặc đổi |
| CV có filter theo ngày | Tab Filters | Filter loại errata sau một ngày sẽ chặn luôn gói bản mới |

**Trường hợp A — CV dùng repo rolling:** sync Library → sửa/tắt filter ngày nếu có → publish CV version mới → promote tới LE thử nghiệm, chờ Capsule sync → trên host `subscription-manager release --unset` (nếu đang khoá), `dnf clean all`, `dnf update -y`, `dnf needs-restarting -r`, reboot theo khung giờ, `subscription-manager facts --update` → kiểm thử rồi promote cùng version lên production và lặp lại cho host production.

**Trường hợp B — CV dùng repo theo minor (EUS):** bật và sync repo của bản minor mới (repo EUS cần subscription EUS trong manifest) → thêm vào CV (hoặc CV riêng nếu một nhóm host cần ở lại minor cũ) → publish, promote → trên host `subscription-manager release --list` phải thấy bản mới, rồi `subscription-manager release --set=<MINOR>`, `dnf clean all`, `dnf update -y`, reboot. Đặt hàng loạt: Hosts → chọn host → **Manage content → System purpose → Release version**.

Lưu ý: với repo rolling không dùng `release --set` sang bản minor mới (danh sách release chỉ gồm các bản minor có repo riêng trong CV); muốn ở lại một minor lâu dài để chỉ nhận vá bảo mật thì dùng EUS, vì RHSA mới thường dựng trên minor mới nhất và incremental update từng errata dễ vỡ phụ thuộc.

## Liên quan

- [Remote execution](remote-execution.md) · [Vá host bằng errata](patching-with-errata.md)
- [Content View và Lifecycle Environment](../content/content-views-and-lifecycle.md) · [Repository và sync](../content/repositories-and-sync.md)
- [Errata và incremental update](../content/errata-and-incremental-update.md)
- [Activation Key chỉ áp dụng lúc đăng ký](activation-key-registration-only.md)
- [RHEL 10: GPG check FAILED, release key 4](../../rhel/rhel10-gpg-release-key-4.md)

## Nguồn

- Red Hat Satellite 6.19, Managing hosts, chương 11 Upgrading hosts to next major Red Hat Enterprise Linux release (PDF tr. 73–75) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing hosts, chương 12 Converting hosts to Red Hat Enterprise Linux (PDF tr. 76–79); mục 1.3–1.4 RHEL lifecycle status (PDF tr. 13–14) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Tài liệu RHEL *Upgrading from RHEL 7 to RHEL 8 / RHEL 8 to RHEL 9 / RHEL 9 to RHEL 10* (Planning, Preparing, Post-upgrade tasks) — tài liệu RHEL trên docs.redhat.com; bảng Inhibitor ở Bước 4 tổng hợp từ các tài liệu này, chưa chạy thử trên lab.
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: bước chuẩn bị host (unset release, update, reboot, cài `leapp-upgrade`), các trường của wizard REX và quy trình nâng minor theo hai trường hợp rolling/EUS lấy từ tài liệu vận hành của dự án; chưa có kết quả chạy Leapp thật trong dự án.
