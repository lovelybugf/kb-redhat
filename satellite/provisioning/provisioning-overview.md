---
title: Tổng quan provisioning trong Satellite
tags: [concept, satellite, provisioning, content-view, activation-key, hammer]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Provisioning hosts, chương 1, 3–8 (PDF tr. 11–56), phụ lục C–D (PDF tr. 154–157); RH403 chương 8–9; kinh nghiệm lab"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Provisioning biến một máy trống thành host RHEL đã cài, đã đăng ký và đã cấu hình, dựa trên bốn khối: nội dung cài đặt, template, tài nguyên mạng và host group. Trước khi build, bấm Resolve ở tab Operating System để chắc host tìm đủ template, và kiểm tra kickstart repo đã sync xong lên Capsule.

## Các phương thức provisioning

| Phương thức | Dùng khi | Điều kiện chính |
|---|---|---|
| Network based (PXE, UEFI HTTP boot) | Máy boot mạng được | DHCP + TFTP (hoặc HTTPBoot) do Capsule quản lý; Anaconda tải kickstart do Satellite render |
| Discovery qua PXE | Không muốn nhập MAC tay, cần pool máy chờ cài | Như trên + Discovery image; tự động hoá bằng discovery rules |
| Discovery không PXE (kexec) | Máy không boot mạng | Cần DHCP; kexec là Technology Preview, không dùng cho production |
| Boot disk | Không có DHCP/TFTP do Satellite quản lý, hoặc NIC/bond không PXE được | ISO chỉ lo phần boot; installer và kickstart vẫn tải từ Satellite/Capsule |
| Image based | VM hoặc cloud instance | Compute resource clone image/template; cấu hình tiếp bằng finish (SSH) hoặc user_data/cloud-init |

- Loại phần cứng: bare-metal (nhận diện bằng MAC), máy ảo qua compute resource (VMware, KVM, OpenShift Virtualization), cloud instance (OpenStack, EC2, GCE, Azure).
- Hệ điều hành đích: RHEL 10, 9, 8; RHEL 7 cần ELS Add-On. Kiến trúc: x86_64, aarch64, ppc64le, s390x — nhưng PXE, Discovery và boot disk chỉ được hỗ trợ trên x86_64.
- Chi tiết từng cách: [DNS/DHCP/TFTP và network boot](dns-dhcp-tftp-and-network-boot.md), [boot disk và kickstart offline](boot-disks-and-offline-kickstart.md), [Discovery](discovery.md), [VMware](compute-resource-vmware.md), [cloud và nền tảng ảo hoá khác](cloud-and-virtualization-providers.md).

## Luồng network boot (rút gọn)

1. Tạo host, chọn domain và subnet; Satellite xin IP từ DHCP Capsule (hoặc từ database tuỳ IPAM).
2. Khi lưu host, Satellite tạo bản ghi DHCP, DNS thuận/ngược và file cấu hình PXELinux/Grub2/iPXE theo MAC trên TFTP Capsule.
3. Máy boot mạng → DHCP trả `next-server` + `filename` → tải boot loader → đọc cấu hình theo MAC → tải kernel/initrd của installer.
4. Installer gọi Satellite/Capsule lấy template `provision` (kickstart) đã render cho host.
5. Anaconda cài OS, `%post` đăng ký bằng Activation Key qua `subscription-manager`, cài client tools, báo "built".
6. Cấu hình PXE của host chuyển về "boot đĩa cục bộ"; máy khởi động lại vào OS mới.

Máy không có trong inventory sẽ nhận menu mặc định: boot đĩa cục bộ, hoặc vào Discovery nếu đã đặt Discovery làm mặc định.

## Nội dung cài đặt

| Nguồn | Khi dùng | Cách khai báo |
|---|---|---|
| Synced Content (kickstart repo) | RHEL — cách khuyến nghị | Bật repo "BaseOS Kickstart" và "AppStream Kickstart", sync, thêm vào Content View, publish/promote; ở host/host group chọn Media Selection = Synced Content |
| Installation media | Hệ điều hành ngoài Red Hat, mirror riêng | Hosts > Provisioning Setup > Installation Media; đường dẫn là cây cài đặt qua HTTP, dùng được biến `$arch`, `$version`, `$major`, `$minor` |
| Image Builder (TAR) | Cài từ image dựng sẵn bằng Anaconda `liveimg` | Đưa file TAR lên file repo hoặc web server; đặt tham số host group `kickstart_liveimg` = URL đầy đủ hoặc tên file |

```bash
# Xem kickstart tree / installation media mà tổ chức thấy được
hammer medium list --organization "<ORG>"

# Tạo installation media cho mirror riêng
hammer medium create --name "<MEDIUM_NAME>" --os-family "Redhat" \
  --path 'http://<MIRROR_FQDN>/rhel/$version/$arch/os/' \
  --organizations "<ORG>" --locations "<LOCATION>"
```

- Môi trường disconnected: nhập kickstart repo từ DVD binary của RHEL (xem [đồng bộ repo](../content/repositories-and-sync.md)).
- Kickstart repo chỉ dùng trong lúc cài; sau khi đăng ký, host dùng các repo RPM thường trong Content View.
- Image Builder image tối thiểu phải có: NetworkManager, authselect, chrony, dnf, dracut, efibootmgr, firewalld, grub2, grub2-efi, grub2-tools-extra, iproute, iputils, kernel, kernel-core, lvm2, subscription-manager, wget (thêm insights-client nếu dùng Lightspeed).

## Template

Không sửa template mặc định (bị khoá, và còn nhận bản cập nhật từ Red Hat); muốn đổi thì Clone rồi sửa bản sao. Bản clone của template "supported" sẽ thành "unsupported". Template viết bằng ERB, xem tại Hosts > Templates > Provisioning Templates.

| Loại (kind) | Vai trò | Template RHEL hay dùng |
|---|---|---|
| Provision | Kickstart chính | Kickstart default |
| PXELinux / PXEGrub2 | Cấu hình boot loader trên TFTP; PXELinux cho BIOS, PXEGrub2 cho UEFI | Kickstart default PXELinux / PXEGrub2 |
| iPXE | Thay PXELinux khi boot iPXE (đã deprecated) | Kickstart default iPXE |
| Finish | Script chạy qua SSH sau image-based provisioning, khi nền tảng không hỗ trợ user_data | Kickstart default finish |
| user_data | Dữ liệu seed cho cloud-init/ignition trên cloud/ảo hoá | Kickstart default user data |
| cloud_init | Image tự tải template từ Satellite qua HTTP(S); không chạy sau NAT | CloudInit default |
| Bootdisk, Kernel Execution (kexec) | Boot không PXE; kexec là Technology Preview | — |
| Snippet | Đoạn ERB dùng lại trong template khác | `create_users`, `subscription_manager_registration`... |

- Chạy code riêng mà không clone template: tạo snippet tên `<tên template> custom pre` hoặc `<tên template> custom post` (ví dụ `Kickstart default custom post`), loại Snippet.
- Đẩy SSH key: snippet `create_users` trong `%post` thêm SSH key của user/user group ở trường "Owned by" của host.
- Partition table nằm ở Hosts > Templates > Partition Tables; mặc định dùng `Kickstart default`. Có sẵn `Kickstart default encrypted` (LUKS; tham số host `disk_enc_passphrase` hoặc `disk_enc_tang_servers` cho Clevis/Tang). Dòng đầu `#Dynamic` biến bảng thành script `%pre` sinh `/tmp/diskpart.cfg` theo RAM/đĩa thực tế.
- Template phải được gắn với Operating System (tab Association của template) và chọn làm mặc định ở tab Templates của OS:

```bash
hammer template list --search 'name ~ "Kickstart default"'
hammer template add-operatingsystem --id <TEMPLATE_ID> --operatingsystem-id <OS_ID>
hammer os set-default-template --id <OS_ID> --config-template-id <TEMPLATE_ID>
hammer os info --id <OS_ID>
```

Bản ghi Operating System cho RHEL được tạo tự động khi bật và sync repo Red Hat; nó gom kiến trúc, partition table, media, template và thuật toán băm mật khẩu root.

## Provisioning context

Organization + Location quyết định tài nguyên nào hiện ra khi tạo host. Domain, subnet, media, template, partition table, compute resource, Capsule, host group và Activation Key phải cùng thuộc org/location đang chọn; thiếu một cái là danh sách tương ứng trống. Trên CLI luôn kèm `--organization` và `--location`.

## Host group cho provisioning — checklist

Cấu hình dưới đây đã chạy thành công trong lab (Satellite 6.19, host RHEL 9, máy EFI):

| Tab | Trường | Giá trị |
|---|---|---|
| Host Group | Lifecycle Environment, Content View | `<LIFECYCLE_ENV>`, `<CONTENT_VIEW>` — CV phải chứa kickstart repo |
| Host Group | Content Source | Capsule phục vụ subnet của host |
| Network | Domain, IPv4 Subnet | `<DOMAIN>`, `<SUBNET>` |
| Operating System | Architecture, Operating System | x86_64, RHEL 9.x |
| Operating System | Media Selection | Synced Content (chọn kickstart tree tương ứng) |
| Operating System | Partition Table | Kickstart default |
| Operating System | PXE Loader | Grub2 UEFI (máy UEFI) hoặc PXELinux BIOS (máy BIOS) |
| Parameters | `kt_activation_keys` | `<ACTIVATION_KEY>` (cùng org; nhiều key ngăn bằng dấu phẩy) |
| Ansible Roles | (tuỳ chọn) | Role chạy tự động sau khi host đăng ký |

Sau khi điền, ở form tạo host bấm **Resolve** cạnh "Provisioning templates": mỗi loại (provision, PXEGrub2/PXELinux, finish...) phải hiện tên template. Loại nào trống thì gắn template đó với OS rồi Resolve lại. Chi tiết host group và tham số: [host group và tham số](../hosts/host-groups-and-parameters.md).

Lưu ý thực tế:
- Kickstart repo phải sync xong lên Capsule (Capsule có Lifecycle Environment chứa CV) trước khi build, nếu không Anaconda báo không tìm thấy cây cài đặt.
- Repo `on_demand` làm lần cài đầu chậm vì Capsule phải kéo gói ngược lên Satellite/CDN — xem [download policy](../content/download-policy.md).
- Activation Key chỉ áp dụng lúc đăng ký; đổi CV/LE về sau xem [AK chỉ áp dụng lúc đăng ký](../hosts/activation-key-registration-only.md).

## Thứ tự kế thừa tham số

Từ thấp đến cao (mức sau ghi đè mức trước): Global → Organization → Location → Domain → Subnet → Operating System → Host group → Host. Tham số hay dùng khi provisioning: `kt_activation_keys`, `kickstart_liveimg`, `fips_enabled`, `disk_enc_passphrase`, `disk_enc_tang_servers`, `autopart_options`.

## Bảo mật khi provisioning

- **Token build:** mỗi host ở build mode có token riêng trong URL kickstart, mặc định sống 360 phút (Administer > Settings > Provisioning > Token duration). Hết hạn thì installer nhận 404. Đặt 0 là tắt token — không nên, vì kẻ giả IP có thể tải kickstart kèm hash mật khẩu root. Cấp token mới: trên trang host chọn Cancel build rồi Build lại.
- **Mật khẩu root mặc định dạng hash:** Administer > Settings > Provisioning > Root password; host group kế thừa được. Tạo hash:

```bash
python3 -c 'import crypt,getpass;pw=getpass.getpass(); print(crypt.crypt(pw)) if (pw==getpass.getpass("Confirm: ")) else exit()'
```

- **FIPS:** đặt thuật toán băm SHA256 cho từng OS rồi bật tham số `fips_enabled` cho host group; kiểm tra trên host bằng `cat /proc/sys/crypto/fips_enabled` (kết quả 1).

```bash
hammer os update --title "<OS_TITLE>" --password-hash SHA256
hammer hostgroup set-parameter --hostgroup "<HOST_GROUP>" --name fips_enabled --value "true"
```

- Quyền tối thiểu cho người provisioning (không cần admin) liệt kê ở phụ lục D; xem [user, role, org](../security/users-roles-orgs.md).

## Lưu ý khác

- Mặc định xoá host **không** xoá VM trên compute resource; muốn xoá kèm thì bật "Destroy associated VM on host delete" (Administer > Settings > Provisioning).
- Hypervisor/container host báo hàng nghìn NIC làm Satellite chậm: lọc bằng "Ignore interfaces with matching identifier" (Settings > Facts) rồi dọn bằng `foreman-rake interfaces:clean`, `foreman-rake facts:clean`.
- Compute profile có sẵn 1-Small, 2-Medium, 3-Large; tạo thêm ở Infrastructure > Compute Profiles.

## Liên quan

- [DNS, DHCP, TFTP và network boot](dns-dhcp-tftp-and-network-boot.md)
- [Boot disk và kickstart khi không có DHCP/TFTP](boot-disks-and-offline-kickstart.md)
- [Discovery](discovery.md) · [Compute resource VMware](compute-resource-vmware.md) · [Cloud và ảo hoá khác](cloud-and-virtualization-providers.md)
- [Content View và Lifecycle](../content/content-views-and-lifecycle.md) · [Đăng ký host](../hosts/host-registration.md)
- [Cổng mạng](../architecture/network-ports.md)

## Nguồn

- Red Hat Satellite 6.19, Provisioning hosts, chương 1 Introduction to provisioning (PDF tr. 11–13) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Provisioning hosts, chương 3 Preparing provisioning content (PDF tr. 24–28), chương 4 Preparing client platforms (PDF tr. 29–34) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Provisioning hosts, chương 5 Preparing templates for provisioning (PDF tr. 35–44), chương 6 Security settings for provisioning (PDF tr. 45–47), chương 7 Provisioning contexts (PDF tr. 48–49) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Provisioning hosts, mục 8.1 Network boot provisioning workflow và 8.5 Creating hosts with PXE boot (PDF tr. 50–56); phụ lục C Host parameter hierarchy, D Permissions (PDF tr. 154–157) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- RH403 (Satellite 6.15), chương 8 Prepare network resources for host provisioning, chương 9 Provision content hosts — cách giải thích loại template và host group.
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: checklist host group (LE, CV có kickstart repo, Content Source = Capsule, Synced Content, Kickstart default, Grub2 UEFI, Resolve) đã dùng để cài thành công host RHEL 9; kickstart repo phải sync lên Capsule trước; repo on_demand làm lần cài đầu chậm.
