---
title: Compute resource VMware vSphere và xử lý lỗi
tags: [howto, satellite, provisioning, hammer, certificate, networking]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Provisioning hosts, chương 12 Provisioning virtual machines in VMware (PDF tr. 96–110), mục 4.1, 4.9–4.10 (PDF tr. 29–34), phụ lục A (PDF tr. 143); RH403 chương 9; kinh nghiệm lab"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Thêm vCenter làm compute resource bằng một user riêng đủ quyền, rồi khai báo compute profile (CPU, RAM, firmware, cluster, datastore, port group) để Satellite tự tạo VM khi tạo host. Network based cần port group có Capsule quản lý DHCP/TFTP; image based clone từ VMware template và cấu hình tiếp bằng finish (SSH) hoặc cloud-init.

Phần cấu hình lấy từ tài liệu hãng. Mục "Lỗi đã gặp trong lab" ghi rõ từng lỗi đã hay chưa kiểm chứng cách sửa.

## Điều kiện

- vCenter Server 8.0 hoặc 7.0 (hai bản đã được Red Hat kiểm thử).
- Satellite tới vCenter TCP 443; Satellite và vCenter phân giải được tên của nhau.
- Network based: một Capsule quản lý DHCP/TFTP trên port group dùng để provision, và port group đó không có DHCP khác (tránh xung đột). Không có DHCP/TFTP thì dùng [boot disk hoặc kickstart offline](boot-disks-and-offline-kickstart.md).
- Image based: có sẵn VMware template.
- Kickstart content đã sync, có Activation Key, host group đầy đủ ([checklist](provisioning-overview.md#host-group-cho-provisioning--checklist)).

## Quyền của user vCenter

Tạo user riêng thay vì dùng administrator, gán role có các quyền (All Privileges →):

| Nhóm | Quyền |
|---|---|
| Datastore | Allocate Space, Browse datastore, Update Virtual Machine files, Low level file operations |
| Network | Assign Network |
| Resource | Assign virtual machine to resource pool |
| Virtual Machine | Change Config, Interaction, Edit Inventory, Provisioning, Guest Operations (tất cả quyền con) |
| Cryptographic operations (chỉ khi dùng vTPM) | Clone, Encrypt, Encrypt new, Migrate, Register VM, Direct Access |

## Tạo compute resource

Web UI: Infrastructure > Compute Resources > Create Compute Resource → Provider **VMware** → VCenter/Server, User, Password → **Load Datacenters** → chọn Datacenter → kiểm tra trường **Fingerprint** đã tự điền → Display Type (VMRC; VNC không hỗ trợ trên ESXi 6.5 trở lên) → tuỳ chọn mật khẩu console ngẫu nhiên, Enable Caching → tab Locations/Organizations → Submit.

```bash
hammer compute-resource create --provider "Vmware" \
  --name "<CR_NAME>" --description "vCenter <VCENTER_FQDN>" \
  --server "<VCENTER_FQDN>" --user "<VCENTER_USER>" --password "<PASSWORD>" \
  --datacenter "<DATACENTER>" \
  --organizations "<ORG>" --locations "<LOCATION>"
```

Tránh để mật khẩu thật trong lịch sử shell (dùng biến môi trường hoặc nhập qua web UI).

## Compute profile cho VMware

Infrastructure > Compute Profiles → chọn profile (có sẵn 1-Small, 2-Medium, 3-Large hoặc tạo mới) → chọn compute resource VMware:

| Trường | Ghi chú |
|---|---|
| CPUs, Cores per socket, Memory (MiB) | Kích thước VM |
| Firmware | Automatic (mặc định), BIOS, EFI, UEFI Secure Boot — phải khớp PXE loader của host group |
| Cluster, Resource pool, Folder | Vị trí đặt VM |
| Guest OS, Virtual H/W version | Kiểu guest và phiên bản phần cứng ảo |
| Memory/CPU hot add, CD-ROM, Boot order | Network phải đứng trước disk khi network based |
| Image | Chọn khi image based |
| SCSI controller, Eager zero, ổ đĩa (datastore, size, thin) | Mặc định lazy zero thick nếu không chọn Eager zero |
| Virtual TPM | Chỉ dùng với firmware UEFI |
| Network Interfaces | Ít nhất một NIC trên mạng do Capsule quản lý |

```bash
hammer compute-profile create --name "<PROFILE>"
hammer compute-profile values create --compute-profile "<PROFILE>" \
  --compute-resource "<CR_NAME>" \
  --compute-attributes "cpus=2,corespersocket=1,memory_mb=4096,cluster=<CLUSTER>,path=<VM_FOLDER>,start=true" \
  --interface "compute_type=VirtualE1000,compute_network=<PORTGROUP>" \
  --volume "size_gb=40G,datastore=<DATASTORE>,name=<DISK_NAME>,thin=true"
```

`compute_type` chọn theo loại NIC cần dùng (ví dụ VirtualVmxnet3 cho RHEL). Đổi giá trị sau này: `hammer compute-profile values update` với cùng tham số.

## Tạo host trên VMware

Hosts > Create Host → Host Group → **Deploy on** = compute resource VMware → **Compute profile** → tab Interfaces: để **trống MAC** (VMware cấp khi tạo VM), giữ Managed/Primary/Provision → tab Operating System: chọn **Provisioning Method** (Network Based, Image Based hoặc Boot disk based) → **Resolve** → tab Virtual Machine kiểm tra giá trị từ compute profile (chọn UEFI Secure Boot nếu cần) → tab Parameters: `kt_activation_keys` → Submit.

Satellite tạo VM, đọc MAC, tạo bản ghi DHCP/DNS/TFTP, bật VM; VM boot mạng và cài như PXE thường.

```bash
# Network based
hammer host create --name "<HOST_NAME>" --hostgroup "<HOST_GROUP>" \
  --organization "<ORG>" --location "<LOCATION>" \
  --compute-resource "<CR_NAME>" --compute-profile "<PROFILE>" \
  --provision-method build --build true --enabled true --managed true \
  --interface "managed=true,primary=true,provision=true,compute_type=VirtualE1000,compute_network=<PORTGROUP>" \
  --volume "size_gb=40G,datastore=<DATASTORE>,name=<DISK_NAME>,thin=true"

# Image based: thay --provision-method image và thêm --image "<IMAGE_NAME>"
```

### Network based hay image based

| | Network based | Image based |
|---|---|---|
| Nguồn hệ điều hành | Kickstart tree qua Capsule | Clone từ VMware template |
| Yêu cầu mạng | DHCP + TFTP (hoặc HTTPBoot) của Capsule trên port group, hoặc boot disk | Finish: DHCP do Capsule quản lý + Satellite SSH được vào VM. Cloud-init: cần DHCP trên subnet |
| Cấu hình sau cài | `%post` của kickstart (đăng ký AK, Ansible role) | Template finish qua SSH, hoặc userdata + cloud-init gọi về Satellite |
| Tốc độ | Chậm hơn (cài từng gói) | Nhanh |
| Bảo trì | Dùng chung template với bare-metal | Phải cập nhật VMware template định kỳ |

### Image based với cloud-init

1. Gắn template `CloudInit default` và `UserData open-vm-tools` vào OS (tab Association), rồi ở OS → tab Templates chọn chúng cho Cloud-init template và User data template.
2. Chuẩn bị VM gốc, chạy bằng root:

```bash
dnf install cloud-init open-vm-tools perl-interpreter perl-File-Temp   # RHEL 9 thêm dbus-tools
cat << EOM > /etc/cloud/cloud.cfg.d/01_network.cfg
network:
  config: disabled
EOM
cat << EOM > /etc/cloud/cloud.cfg.d/10_datasource.cfg
datasource_list: [NoCloud]
datasource:
  NoCloud:
    seedfrom: https://<CAPSULE_FQDN>:9090/userdata/
EOM
wget -O /etc/pki/ca-trust/source/anchors/cloud-init-ca.crt https://<CAPSULE_FQDN>/pub/katello-server-ca.crt
update-ca-trust extract
```

   Provision thẳng từ Satellite thì `seedfrom` là `https://<SATELLITE_FQDN>/userdata/` và CA lấy từ Satellite. Cấu hình module trong `/etc/cloud/cloud.cfg` theo mục 12.14 (PDF tr. 106–108).
3. Dọn VM trước khi chuyển thành template: dừng rsyslog/auditd, `dnf remove --oldinstallonly`, xoá log cũ, `/etc/udev/rules.d/70*`, file ifcfg cũ, `/etc/ssh/ssh_host_*`, `~root/.ssh/known_hosts`, lịch sử shell.
4. Chuyển VM thành template, khai báo image trong compute resource (tick User data):

```bash
hammer compute-resource image create --compute-resource "<CR_NAME>" \
  --name "<IMAGE_NAME>" --operatingsystem "<OS_TITLE>" --architecture "x86_64" \
  --username root --user-data true --uuid "<TEMPLATE_PATH>"
```

`--uuid` là đường dẫn tương đối của template trong vSphere. Với cloud-init, VM luôn gọi về Satellite (cổng 80 rồi chuyển 443) kể cả khi đăng ký qua Capsule — firewall phải cho phép; Capsule phải nằm trong danh sách trusted proxies của Satellite.

## Vận hành

- **Cache:** bật Enable caching để trang VMware load nhanh; khi vCenter thay đổi (port group, template mới) bấm **Refresh Cache**, hoặc gọi API `PUT https://<SATELLITE_FQDN>/api/compute_resources/<CR_ID>/refresh_cache` (lấy ID bằng `hammer compute-resource list`).
- **Import VM có sẵn:** compute resource → tab Virtual Machines → Import as managed Host / Import as unmanaged Host.
- **Xoá:** xoá host không xoá VM, trừ khi bật "Destroy associated VM on host delete" (Settings > Provisioning). Xoá VM ở tab Virtual Machines thì host vẫn còn, phải xoá tay — xem [xoá host](../hosts/host-deletion-and-unregister.md).
- **Console trong trình duyệt (noVNC):** Display type VNC, mở 5900–5930/tcp trên Satellite, import CA của Satellite vào trình duyệt (thêm ngoại lệ bảo mật là không đủ).

## Lỗi đã gặp trong lab

**1. Fingerprint public key của vCenter không khớp — đã kiểm chứng cách sửa.** Khi tạo/sửa compute resource, Satellite báo vCenter trình ra public key có hash khác với hash đang lưu (thường do vCenter đổi certificate hoặc trường Fingerprint bị nhập sai). Xử lý: xác nhận certificate vCenter là thật (hỏi đội VMware), rồi đặt Fingerprint đúng bằng giá trị vCenter trả về trong thông báo lỗi (hoặc bấm Load Datacenters để tự điền lại) → tạo được compute resource.

**2. Liệt kê network báo `NoMethodError` liên quan `_ref` — chưa kiểm chứng.** Lỗi xuất hiện khi Satellite đọc danh sách network/port group. Đã thử truyền thẳng ID port group dạng `dvportgroup-<ID>` vào `compute_network` qua hammer, kết quả chưa rõ. Hướng kiểm tra thêm: Refresh Cache, quyền đọc của user trên distributed switch/port group, xem `/var/log/foreman/production.log` để biết đối tượng nào bị nil.

**3. VM phần cứng ảo v21 không boot được ở BIOS — chưa có cách sửa.** Chuyển firmware sang BIOS thì vCenter báo "ACPI motherboard layout requires EFI"; thêm `chipset.motherboardLayout=i440bx` không có tác dụng. Ảnh hưởng: boot disk chỉ chạy BIOS (Host image iPXE) không dùng được. Hướng đi: giữ EFI và dùng Grub2 UEFI, Full host image hoặc kickstart OEMDRV; chọn Virtual H/W version thấp hơn trong compute profile là phương án chưa thử.

**4. Tạo host báo lỗi DNS proxy 400** khi mạng không có DNS do Satellite quản lý — xem [lỗi DNS khi tạo host](dns-dhcp-tftp-and-network-boot.md#lỗi-dns-khi-tạo-host).

Lỗi thường gặp khác (theo tài liệu hãng): VM được tạo nhưng không boot mạng → port group còn DHCP khác, boot order chưa đặt network trước, hoặc firmware VM không khớp PXE loader (EFI ↔ Grub2 UEFI, BIOS ↔ PXELinux BIOS).

## Liên quan

- [Tổng quan provisioning](provisioning-overview.md) · [DNS, DHCP, TFTP và network boot](dns-dhcp-tftp-and-network-boot.md)
- [Boot disk và kickstart khi không có DHCP/TFTP](boot-disks-and-offline-kickstart.md)
- [KVM, OpenShift Virtualization và cloud](cloud-and-virtualization-providers.md) · [virt-who](../content/virt-who.md)
- [Certificate](../installation/certificates.md)

## Nguồn

- Red Hat Satellite 6.19, Provisioning hosts, chương 12 Provisioning virtual machines in VMware, mục 12.1–12.10 (PDF tr. 96–103) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Provisioning hosts, chương 12, mục 12.11–12.17 cloud-init, xoá/import VM, cache (PDF tr. 104–110) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Provisioning hosts, mục 4.1 và 4.9–4.10 Compute profiles (PDF tr. 29–34), mục 2.4 Prerequisites for image-based provisioning (PDF tr. 18), phụ lục A Using noVNC (PDF tr. 143) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- RH403 (Satellite 6.15), chương 9 Provision content hosts — compute resource và compute profile.
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: sửa lỗi fingerprint vCenter bằng giá trị vCenter trả về (đã kiểm chứng); lỗi `NoMethodError _ref` khi liệt kê network (chưa có cách sửa xác nhận); VM HW v21 không boot BIOS, `chipset.motherboardLayout=i440bx` không giúp.
