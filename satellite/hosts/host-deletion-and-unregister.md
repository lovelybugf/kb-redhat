---
title: Huỷ đăng ký và xoá host khỏi Satellite (unregister, delete, VM)
tags: [concept, satellite, registration, provisioning, hammer]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Administering, Phụ lục F.7, F.11 (PDF tr. 140–142, 148–151); Managing hosts, mục 1.1, 2.2, 4.3–4.4 (PDF tr. 12, 19, 27–28); kiểm chứng trên lab Satellite 6.19 (ẩn danh), 2026"
reliability: internal-tested
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** `subscription-manager unregister` chỉ huỷ đăng ký; bản ghi host vẫn nằm lại trên Satellite trừ khi bật setting toàn hệ thống Delete Host upon unregister (`unregister_delete_host`, mặc định No). Xoá host trên Satellite không xoá VM (trừ host gắn Compute Resource khi bật Destroy associated VM on host delete), còn xoá VM thì bản ghi host vẫn còn và vẫn hiện trong báo cáo CVE.

## Ba đối tượng khác nhau

| Đối tượng | Nằm ở đâu | Mất đi khi |
|---|---|---|
| Bản ghi host (ID, lịch sử errata, báo cáo OpenSCAP, tham số, host group, audit) | Database Satellite | Xoá host trên Satellite, hoặc unregister khi `unregister_delete_host = true` |
| Đăng ký (UUID, chứng chỉ consumer, Registered To/At, CV/LE) | Máy client + phần subscription của bản ghi host | `subscription-manager unregister`, hoặc xoá host |
| Máy ảo | Hypervisor/vCenter | Xoá trên hypervisor, hoặc xoá host có Compute Resource khi `destroy_vm_on_host_delete = true` |

Hiểu ba đối tượng này tách rời thì sẽ tránh được hai lỗi hay gặp: host "ma" còn trên Satellite sau khi máy đã huỷ, và xoá nhầm VM khi chỉ muốn dọn bản ghi.

## Thao tác nào tác động tới đâu

| Thao tác | Bản ghi host | Đăng ký | VM | Kiểm chứng |
|---|---|---|---|---|
| `subscription-manager unregister`, setting để No | Còn, giữ nguyên ID và lịch sử; mất UUID, Registered To/At, Content View | Mất; máy báo *System has been unregistered*, `dnf repolist` trống | Không đổi | Lab 6.19 |
| `subscription-manager unregister`, setting bật Yes | Bị xoá ngay; audit ghi `destroy Host::Base <HOST_FQDN>`, IP nguồn là Capsule mà host đăng ký qua | Mất | Host không gắn Compute Resource: không đổi | Lab 6.19 |
| Xoá host trên Satellite (host không gắn Compute Resource) | Bị xoá vĩnh viễn | Phần phía Satellite mất | VM vẫn chạy | Lab 6.19 |
| Xoá VM trên vCenter, không đụng Satellite | **Vẫn còn**, vẫn hiện trong báo cáo CVE/errata | Vẫn coi như đăng ký, chỉ không check-in nữa | Mất | Lab 6.19 |
| Xoá host gắn Compute Resource, `destroy_vm_on_host_delete = true` | Bị xoá | Mất | Bị xoá trên Compute Resource | Theo tài liệu, chưa test |
| Disassociate host (Hosts → All Hosts → chọn → Disassociate hosts) | Còn | Không đổi | Còn, chỉ bỏ liên kết với Compute Resource | Theo tài liệu |

Đăng ký lại sau unregister: setting để No thì máy dùng lại đúng bản ghi cũ (cùng ID, UUID mới); setting Yes thì đăng ký lại sinh host mới, ID mới, lịch sử cũ mất (lab 6.19).

## Hai setting liên quan

| Tên trên Web UI | Tên hammer/API | Tab (Administer → Settings) | Mặc định | Khi bật Yes |
|---|---|---|---|---|
| Delete Host upon unregister | `unregister_delete_host` | Content | No | Máy unregister thì Satellite xoá luôn bản ghi host; tài liệu cảnh báo tài nguyên gắn với host (VM, bản ghi DNS) cũng có thể bị xoá theo |
| Destroy associated VM on host delete | `destroy_vm_on_host_delete` | Provisioning | No | Xoá host thì xoá VM trên Compute Resource; để No thì VM chỉ bị gỡ liên kết, vẫn còn, không bị tắt, có thể associate lại |

Cả hai là setting **toàn Satellite**, không đặt riêng theo organization hay location được.

### Bật/tắt trên Web UI (đã làm trên lab)

1. **Administer → Settings**, gõ `unregister` vào ô Search (hoặc mở tab **Content**, tìm dòng *Delete Host upon unregister*).
2. Bấm biểu tượng bút chì ở cột Value → chọn **Yes** hoặc **No** → bấm dấu ✓ để lưu.
3. Giá trị có hiệu lực ngay, không cần khởi động lại dịch vụ.

Setting *Destroy associated VM on host delete* chỉnh cùng cách ở tab **Provisioning** (tìm `destroy`).

Hammer tương đương (lệnh `set` cho `unregister_delete_host` đã dùng trong testcase lab để bật/tắt setting; xem lại bằng `info` sau khi đặt):

```bash
hammer settings info --name unregister_delete_host
hammer settings set --name unregister_delete_host --value true
hammer settings info --name destroy_vm_on_host_delete
```

## Chọn để No hay bật Yes

| | Để No (mặc định) | Bật Yes |
|---|---|---|
| Ưu điểm | Giữ lịch sử host (errata, OpenSCAP, tham số, host group); đăng ký lại dùng lại bản ghi | Satellite tự dọn host khi máy unregister, không cần bước xoá tay |
| Nhược điểm | Phải thêm bước xoá host vào quy trình huỷ máy và rà host sót định kỳ | Unregister để sửa lỗi cũng làm mất lịch sử; áp cho mọi organization; host có Compute Resource có thể kéo theo VM/DNS |
| Hợp với | Môi trường cần lưu vết, hay đăng ký lại host | Môi trường máy sinh/huỷ thường xuyên, quy trình huỷ máy chuẩn hoá |

Khuyến nghị: giữ *Destroy associated VM on host delete* = No để thao tác dọn bản ghi trên Satellite không bao giờ xoá nhầm VM.

## Quy trình huỷ máy (decommission)

1. Trên máy, trước khi xoá VM (chạy bằng root):

```bash
subscription-manager unregister
insights-client --unregister
```

   Bỏ dòng `insights-client` nếu host không dùng Lightspeed.

2. Trên Satellite, nếu setting để No: xoá bản ghi host.
   - Web UI: **Hosts → All Hosts** → tick host → menu tuỳ chọn → **Delete** → tick *I understand that this action cannot be undone* → **Delete**.
   - Hammer:

```bash
hammer host delete --name "<HOST_FQDN>" --organization "<ORG>" --location "<LOCATION>"
```

3. Xoá VM trên hypervisor (hoặc để Satellite xoá nếu host gắn Compute Resource và đã chủ đích bật `destroy_vm_on_host_delete`).
4. Kiểm tra như mục dưới.

Nếu VM bị xoá trước mà quên unregister: bản ghi host vẫn còn và tiếp tục làm sai số liệu báo cáo CVE/errata cho tới khi xoá tay ở bước 2.

## Tìm host còn sót và host đã bị xoá

| Muốn biết | Web UI | Ghi chú |
|---|---|---|
| Host không còn đăng ký nhưng bản ghi vẫn còn | **Hosts → All Hosts**, Search `null? subscription_uuid` | Danh sách luôn có chính Satellite Server và host tạo từ provisioning nhưng chưa cài; các host khác là máy đã unregister mà chưa xoá |
| Host lâu không check-in (VM có thể đã bị xoá) | **Hosts → All Hosts**, Search `last_checkin < "7 days ago"` | Đối chiếu với hypervisor trước khi xoá |
| Host bị xoá lúc nào, từ đâu | **Monitor → Audits**, Search `type = host and action = destroy` | Xoá do unregister hiện user nội bộ *Anonymous Admin* và IP của Capsule nhận lệnh |

Hammer tương đương:

```bash
hammer host list --search 'null? subscription_uuid' --fields Id,Name
hammer audit list --search 'type = host and action = destroy'
```

## Lưu ý

- **Không xoá chính Satellite Server** khỏi danh sách host, kể cả khi nó không tự đăng ký (Managing hosts mục 1.1).
- Xoá host là vĩnh viễn: mất lịch sử errata, báo cáo OpenSCAP, gán policy, tham số của host. Cần giữ thì chỉ unregister, hoặc đăng ký lại để dùng lại bản ghi cũ.
- Chuyển host sang organization khác: unregister trước, đổi organization (Hosts → All Hosts → Change associations → Organization, chọn *Fix on mismatch*), rồi đăng ký lại.
- Đăng ký lại một host đang chạy: xem [Huỷ đăng ký và đăng ký lại host](host-reregistration.md) (Force, dọn CA/proxy, `hammer host delete`).

## Liên quan

- [Huỷ đăng ký và đăng ký lại host](host-reregistration.md)
- [Đăng ký host bằng global registration](host-registration.md)
- [Báo cáo template và cảnh báo CVE](report-templates-and-cve-alert.md)
- [Compute Resource VMware](../provisioning/compute-resource-vmware.md)
- [Tắt Lightspeed](disable-lightspeed.md)

## Nguồn

- Red Hat Satellite 6.19, Administering Red Hat Satellite, Phụ lục F.7 Content management settings — *Delete Host upon unregister* (PDF tr. 140–142) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Administering Red Hat Satellite, Phụ lục F.11 Host provisioning settings — *Destroy associated VM on host delete* (PDF tr. 148–151) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing hosts, mục 4.3–4.4 Removing a host from Satellite (PDF tr. 27–28); mục 2.2–2.3 disassociate VM, đổi organization (PDF tr. 19–20); mục 1.1 cảnh báo không xoá Satellite Server (PDF tr. 12) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026, lab Satellite 6.19 có Capsule: unregister với setting No (host còn, chỉ mất phần subscription) và Yes (host bị xoá ngay, audit `destroy Host::Base`, request từ Capsule); đăng ký lại dùng lại bản ghi (No) hoặc tạo host mới (Yes); xoá host không gắn Compute Resource thì VM vẫn chạy; xoá VM trên vCenter thì host còn trong báo cáo CVE; các câu search `null? subscription_uuid` và `type = host and action = destroy`. Bật/tắt `unregister_delete_host` bằng `hammer settings set` đã dùng trong testcase. Chưa test: `destroy_vm_on_host_delete`, xoá VM/DNS kèm theo khi bật Yes cho host có Compute Resource.
