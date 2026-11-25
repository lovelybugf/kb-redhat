---
title: Boot disk và cài kickstart khi không có DHCP/TFTP
tags: [howto, satellite, provisioning, capsule, rhel, download-policy]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Provisioning hosts, chương 9 (PDF tr. 67–70), mục 6.1 (PDF tr. 45), mục 8.1 (PDF tr. 50–52); RHEL 9 Automatically installing RHEL (OEMDRV); kinh nghiệm lab"
reliability: internal-tested
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Không có DHCP/TFTP do Satellite quản lý thì cài host bằng boot disk, hoặc như cách đã thành công trong lab: boot DVD RHEL kèm ISO nhãn `OEMDRV` chứa kickstart do Satellite render. Dù boot kiểu nào, kickstart vẫn tải gói từ Capsule và `%post` đăng ký bằng Activation Key, nên Capsule phải có kickstart repo đã sync và token build còn hạn.

Mức kiểm chứng trong trang: Cách A (DVD + OEMDRV) **đã chạy thành công** trong lab Satellite 6.19 / RHEL 9. Cách B (gõ `inst.ks` ở GRUB) mới kiểm tra URL trả HTTP 200, **chưa boot thật**. Full host image đã tạo và gắn cho máy EFI nhưng **chưa thấy kết quả cài**. Phần loại boot disk và lệnh `hammer bootdisk` lấy từ tài liệu hãng.

## Các loại boot disk

| Loại | Nội dung | Cần DHCP | Ghi chú |
|---|---|---|---|
| Full host image | Kernel + initrd của installer cho đúng một host, boot loader nạp installer ngay từ ISO | Không bắt buộc | Dựa trên SYSLINUX + GRUB2, hợp đa số NIC; chứa token build nên hết hạn theo token. Lab: ISO khoảng 240 MB, dùng được cho máy EFI |
| Subnet image | ISO chung cho mọi host cùng subnet (iPXE), chainload installer từ TFTP Capsule, gửi MAC để Capsule khớp host | Có | Cần gói `ipxe-bootimgs` trên Capsule; chỉ một số NIC được iPXE hỗ trợ |
| Host image (iPXE nhỏ) | ISO iPXE khoảng 1 MB cho một host | — | Tài liệu 6.19 không mô tả. Lab: chỉ boot được BIOS, mặc định bị tắt trong menu tải boot disk |

Luồng chung: tạo host trong Satellite → tạo ISO → boot máy từ ISO → máy gọi Templates Capsule lấy kickstart và cài → cuối cài đặt đăng ký bằng Activation Key, cài client tools từ repo Red Hat Satellite Client 6.

Điều kiện: có Activation Key cho host; repo Red Hat Satellite Client 6 đúng phiên bản OS đã sync và bật trong Activation Key; host group đầy đủ như [checklist trong trang tổng quan](provisioning-overview.md#host-group-cho-provisioning--checklist).

## Tạo boot disk

Web UI: tạo host (Hosts > Create Host, chọn host group, kiểm tra interface Managed/Primary/Provision, bấm Resolve, thêm `kt_activation_keys`) → trang chi tiết host → menu **⋮** → `Full host '<HOST_FQDN>' image`. Subnet image: Infrastructure > Subnets → Actions → Subnet generic image.

```bash
hammer host create --name "<HOST_NAME>" --hostgroup "<HOST_GROUP>" \
  --organization "<ORG>" --location "<LOCATION>" --mac "<MAC>" \
  --build true --enabled true --managed true
hammer host interface update --host "<HOST_FQDN>" --managed true --primary true --provision true

hammer bootdisk host --host "<HOST_FQDN>" --full true     # Full host image
hammer bootdisk subnet --subnet "<SUBNET>"                # Subnet image
```

Boot disk cũng là cách vượt qua bond NIC cần cấu hình phía switch (không PXE được): sau khi kernel installer nạp, bond hoạt động bình thường.

## Token build và hết hạn

- Host ở build mode có token riêng gắn trong URL kickstart; mặc định sống 360 phút (Administer > Settings > Provisioning > Token duration). Lab giữ mức 6 giờ.
- Token hết hạn → installer tải kickstart nhận 404. Full host image nhúng token nên ISO cũ cũng hết dùng được.
- Cấp token mới (đã làm trong lab): trang host → **⋮ › Cancel build**, rồi **⋮ › Build**. Sau đó tải lại ISO hoặc render lại kickstart.

## Cách A — DVD RHEL + ISO OEMDRV (đã chạy thành công)

Nguyên lý: khi không có tham số `inst.ks`, Anaconda tự tìm volume có nhãn `OEMDRV` và nạp file `ks.cfg` ở gốc volume đó (thấy trong log dracut `parse-kickstart`). DVD RHEL lo phần boot, kickstart do Satellite render lo phần còn lại.

1. Chuẩn bị host group và tạo host ở build mode như trên. Mạng không có DNS do Satellite quản lý mà tạo host báo lỗi DNS proxy 400 → bỏ tick **Managed** trên interface (xem [lỗi DNS khi tạo host](dns-dhcp-tftp-and-network-boot.md#lỗi-dns-khi-tạo-host)).
2. Lấy kickstart đã render cho host — một trong hai cách:

```bash
# Qua API Satellite (lab đã dùng); curl sẽ hỏi mật khẩu
curl -s -u <USER> "https://<SATELLITE_FQDN>/api/hosts/<HOST_ID>/template/provision" \
  | python3 -c 'import json,sys; print(json.load(sys.stdin)["template"])' > ks.cfg

# Qua URL unattended của Capsule (URL có token nằm trong template PXE đã render của host)
curl -s -o ks.cfg "http://<CAPSULE_FQDN>:8000/unattended/provision?token=<TOKEN>"
```

3. Mở `ks.cfg` kiểm tra: dòng `url`/`repo` trỏ về Capsule (đúng Content Source), có đoạn `subscription-manager register` với Activation Key. Subnet không có DHCP thì sửa dòng `network` thành IP tĩnh (lab đã thêm tay):

```text
network --bootproto=static --device=<NIC> --ip=<IP> --netmask=<NETMASK> --gateway=<GATEWAY> --nameserver=<DNS_IP> --hostname=<HOST_FQDN> --activate
```

4. Đóng gói thành ISO nhãn `OEMDRV`, file tên đúng `ks.cfg` ở gốc:

```bash
mkdir oemdrv && cp ks.cfg oemdrv/ks.cfg
xorriso -as mkisofs -V OEMDRV -J -r -o <HOST_NAME>-oemdrv.iso oemdrv/
```

   Trên Windows có thể dựng ISO bằng IMAPI2 (COM qua PowerShell) miễn là volume label là `OEMDRV`.

5. Gắn vào VM: CD1 = DVD RHEL (nên cùng bản minor với kickstart tree), CD2 = ISO OEMDRV; firmware EFI. Menu GRUB của DVD mặc định chọn "Test this media & install" — chọn dòng "Install Red Hat Enterprise Linux" để bỏ bước kiểm tra media.
6. Anaconda tự nạp kickstart, tải gói từ Capsule, `%post` đăng ký bằng Activation Key và báo "built" cho Satellite. Sau khi host đăng ký, Ansible role gắn ở host group tự chạy.
7. Dọn dẹp: tháo hai CD; xoá `ks.cfg` và ISO vì chứa hash mật khẩu root, tên Activation Key và token (xem [bí mật trong tài liệu](../security/documentation-secrets.md)).

Kiểm tra sau khi cài:

```bash
# trên host mới
subscription-manager identity
subscription-manager repos --list-enabled
# trên Satellite
hammer host info --name "<HOST_FQDN>"
```

## Cách B — gõ `inst.ks` ở GRUB (chưa boot thật)

Boot từ DVD RHEL, ở menu GRUB bấm `e`, nối vào cuối dòng `linux` (hoặc `linuxefi`):

```text
inst.ks=http://<CAPSULE_IP>:8000/unattended/provision?token=<TOKEN> ip=<IP>::<GATEWAY>:<NETMASK>:<HOST_FQDN>:<NIC>:none nameserver=<DNS_IP>
```

rồi Ctrl+X. Tham số `ip=` cần khi không có DHCP để dracut có mạng tải kickstart; dùng IP Capsule thay FQDN nếu chưa chắc DNS. Lab chỉ mới xác nhận URL trả về HTTP 200:

```bash
curl -s -o /dev/null -w '%{http_code}\n' "http://<CAPSULE_IP>:8000/unattended/provision?token=<TOKEN>"
```

## Luồng mạng khi cài (lab)

Với Content Source = Capsule, máy mới chỉ cần nói chuyện với Capsule:

| Bước | Đích | Cổng |
|---|---|---|
| Tải kickstart, báo built | Capsule (Templates) | 8000/tcp (HTTP) |
| Tải kickstart tree và gói | Capsule (Pulp) | 80/tcp, 443/tcp |
| Đăng ký RHSM, tải client tools | Capsule | 443/tcp |

- Danh sách cổng đầy đủ: [cổng mạng](../architecture/network-ports.md).
- Repo `on_demand`: lần cài đầu chậm vì Capsule phải kéo gói từ Satellite (Satellite lại kéo từ CDN). Cần cài nhanh, ổn định thì cân nhắc `immediate` cho kickstart repo, hoặc cài thử một máy để "làm nóng" cache — xem [download policy](../content/download-policy.md).

## Sự cố thường gặp

| Triệu chứng | Nguyên nhân | Xử lý |
|---|---|---|
| Installer nhận 404 khi tải kickstart | Token build hết hạn | Cancel build → Build, lấy lại ISO/kickstart |
| Tạo host báo lỗi DNS proxy 400 | Domain/subnet gắn DNS Capsule nhưng mạng không có DNS do Satellite quản lý | Bỏ tick Managed trên interface, hoặc bỏ DNS Capsule khỏi domain/subnet |
| Anaconda không tự nạp kickstart | Nhãn volume khác `OEMDRV` hoặc file không tên `ks.cfg` ở gốc | Tạo lại ISO đúng nhãn và tên file |
| Anaconda báo không tìm thấy cây cài đặt | CV thiếu kickstart repo, hoặc Capsule chưa sync Lifecycle Environment | Thêm repo, publish/promote, sync Capsule |
| Cài rất chậm lần đầu | Repo `on_demand` | Như mục luồng mạng |
| VM VMware HW v21 không bật được ở BIOS: "ACPI motherboard layout requires EFI" | HW version mới yêu cầu EFI | Dùng EFI (Cách A hoặc Full host image); `chipset.motherboardLayout=i440bx` không có tác dụng trong lab (chưa rõ nguyên nhân) |
| Không thấy loại Host image (iPXE) khi tải boot disk | Loại này mặc định bị tắt và chỉ chạy BIOS | Dùng Full host image cho máy EFI |

## Liên quan

- [Tổng quan provisioning](provisioning-overview.md) · [DNS, DHCP, TFTP và network boot](dns-dhcp-tftp-and-network-boot.md)
- [Compute resource VMware](compute-resource-vmware.md) · [Discovery không PXE](discovery.md)
- [Download policy](../content/download-policy.md) · [Đăng ký host](../hosts/host-registration.md)

## Nguồn

- Red Hat Satellite 6.19, Provisioning hosts, chương 9 Using boot disks to provision hosts (PDF tr. 67–70) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Provisioning hosts, mục 6.1 Configuring the security token validity duration (PDF tr. 45), phần mở đầu chương 8 về bond NIC và mục 8.1 workflow (PDF tr. 50–52) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- RHEL 9, Automatically installing RHEL, phần đặt kickstart trên volume cục bộ nhãn OEMDRV — https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: cài thành công RHEL 9 bằng DVD + ISO OEMDRV chứa kickstart render từ Satellite (tải gói từ Capsule, đăng ký bằng AK, Ansible role của host group tự chạy); cấp lại token bằng Cancel build/Build; URL `inst.ks` trả HTTP 200 nhưng chưa boot thử; Host image iPXE chỉ chạy BIOS và mặc định tắt; Full host image (~240 MB) chưa thấy kết quả; HW v21 không boot BIOS.
