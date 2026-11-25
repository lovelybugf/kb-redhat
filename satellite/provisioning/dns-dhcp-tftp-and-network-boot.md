---
title: Tích hợp DNS, DHCP, TFTP và network boot (PXE, UEFI HTTP)
tags: [howto, satellite, provisioning, networking, capsule, idm]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Integrating provisioning infrastructure services, chương 1–3 (PDF tr. 7–30); Provisioning hosts, chương 2 và 8, phụ lục E (PDF tr. 14–23, 50–66, 158–167); RH403 chương 8; kinh nghiệm lab"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Capsule tự chạy DNS (BIND), DHCP (ISC), TFTP qua `satellite-installer` hoặc tích hợp dịch vụ sẵn có (DNS RFC 2136, IdM, Infoblox; ISC DHCP từ xa; TFTP qua NFS), rồi gán vào domain, subnet. Chọn PXE loader theo firmware: Grub2 UEFI cho máy UEFI, PXELinux BIOS cho máy BIOS, Grub2 UEFI HTTP khi muốn bỏ TFTP.

## Tài nguyên mạng trong Satellite

- **Domain** (Infrastructure > Domains): host nào cũng phải thuộc một domain, kể cả khi Satellite không quản lý DNS. Trường DNS Capsule quyết định ai tạo bản ghi A/AAAA.
- **Subnet** (Infrastructure > Subnets): khai báo network, mask, gateway, DNS, IPAM, dải IP; tab Capsules gán Capsule cho từng dịch vụ (DHCP, TFTP, Reverse DNS, Discovery, HTTPBoot, Templates, Remote Execution).
- Subnet đã khai báo trong DHCP của Capsule có thể nhập nhanh: Infrastructure > Capsules > Actions > Import IPv4 subnets (mục này không hiện nghĩa là DHCP của Capsule chưa đúng).

```bash
hammer domain create --name "<DOMAIN>" --dns "<CAPSULE_FQDN>" \
  --organizations "<ORG>" --locations "<LOCATION>"

hammer subnet create --name "<SUBNET>" --network "<NETWORK_ADDRESS>" --mask "<NETMASK>" \
  --gateway "<GATEWAY>" --dns-primary "<DNS_IP>" --domains "<DOMAIN>" \
  --ipam DHCP --boot-mode DHCP --from "<RANGE_START>" --to "<RANGE_END>" \
  --dhcp "<CAPSULE_FQDN>" --tftp "<CAPSULE_FQDN>" --dns "<CAPSULE_FQDN>" \
  --organizations "<ORG>" --locations "<LOCATION>"
```

| IPAM | Cách cấp IP | Lưu ý |
|---|---|---|
| DHCP | Capsule tìm IP trống đầu tiên trong dải, ping ICMP/TCP trước khi cấp | Máy tắt hoặc chặn ping bị coi là IP trống |
| Internal DB | IP kế tiếp chưa có trong database Satellite | Không an toàn khi tạo nhiều host song song |
| Random DB | IP ngẫu nhiên chưa có trong database | An toàn khi tạo song song |
| EUI-64 | Sinh IPv6 từ MAC | — |
| External IPAM / None | Hệ thống ngoài / nhập tay | — |

Subnet dùng IPAM DHCP, Internal DB hoặc Random DB phải do Satellite kiểm soát độc quyền; bản ghi DHCP tạo ngoài Satellite sẽ gây xung đột. Nếu dùng cả Discovery, tách dải cho host đã provision và dải cho host đang discover.

Nhiều subnet/zone trên cùng Capsule do installer quản lý: khai báo `dhcp::pools` và `dns::zones` trong `/etc/foreman-installer/custom-hiera.yaml`, chạy lại `satellite-installer`, rồi tạo subnet tương ứng trong Satellite.

## DNS

| Provider | Dùng cho | Xác thực |
|---|---|---|
| `nsupdate` | BIND trên Capsule (installer quản lý hoặc tự quản) hoặc DNS từ xa hỗ trợ RFC 2136 | TSIG key (`/etc/foreman-proxy/rndc.key`) |
| `nsupdate_gss` | DNS của IdM khi Capsule là thành viên domain IdM | GSS-TSIG, keytab Kerberos |
| `infoblox` | Appliance Infoblox | Tài khoản WAPI (DNS Admin, DHCP Admin) |

Các lệnh dưới chạy bằng root trên Satellite hoặc Capsule tương ứng.

**DNS do installer quản lý** (chưa có DNS trong mạng):

```bash
satellite-installer --foreman-proxy-dns true \
  --foreman-proxy-dns-provider nsupdate \
  --foreman-proxy-dns-managed true \
  --foreman-proxy-dns-zone "<DOMAIN>" \
  --foreman-proxy-dns-reverse "<REVERSE_ZONE>" \
  --foreman-proxy-dns-forwarders "<UPSTREAM_DNS_IP>" \
  --reset-foreman-proxy-dns-server
```

**DNS từ xa hỗ trợ RFC 2136** (BIND sẵn có): đặt key vào `/etc/foreman-proxy/rndc.key`, phân quyền, thử bằng `nsupdate`, rồi trỏ Capsule tới:

```bash
chown -v root:foreman-proxy /etc/foreman-proxy/rndc.key
chmod -v 640 /etc/foreman-proxy/rndc.key
restorecon -v /etc/foreman-proxy/rndc.key

# thử thêm rồi xoá một bản ghi test
echo -e "server <DNS_SERVER_IP>\n update add test.<DOMAIN> 3600 IN A <TEST_IP>\n send\n" | nsupdate -k /etc/foreman-proxy/rndc.key
host test.<DOMAIN> <DNS_SERVER_IP>
echo -e "server <DNS_SERVER_IP>\n update delete test.<DOMAIN> 3600 IN A <TEST_IP>\n send\n" | nsupdate -k /etc/foreman-proxy/rndc.key

satellite-installer --foreman-proxy-dns true \
  --foreman-proxy-dns-provider nsupdate \
  --foreman-proxy-dns-managed false \
  --foreman-proxy-dns-server "<DNS_SERVER_IP>" \
  --foreman-proxy-keyfile /etc/foreman-proxy/rndc.key
```

**DNS của IdM:**
- Capsule không thuộc domain IdM → dùng TSIG: trên IdM server thêm `include "/etc/rndc.key";` và khối `controls` cho phép IP Satellite trong `/etc/named.conf`, đặt BIND update policy `grant "rndc-key" zonesub ANY;` cho zone, bật Dynamic update, rồi cấu hình như DNS RFC 2136 ở trên.
- Capsule là thành viên IdM → dùng GSS-TSIG (an toàn hơn, ít quản lý key):

```bash
# trên một máy thuộc domain IdM
kinit admin
ipa service-add capsule/<CAPSULE_FQDN>
ipa dnszone-show <DOMAIN> --all | grep "BIND update policy"
ipa dnszone-mod <DOMAIN> --dynamic-update=TRUE --allow-sync-ptr=TRUE \
  --update-policy="<EXISTING_POLICY> grant capsule\047<CAPSULE_FQDN>@<REALM> wildcard * ANY;"
# làm tương tự cho reverse zone (không cần --allow-sync-ptr)

# trên Capsule (cài ipa-client và ipa-client-install nếu chưa join)
kinit admin
rm --force /etc/foreman-proxy/dns.keytab
ipa-getkeytab -p capsule/<CAPSULE_FQDN>@<REALM> -k /etc/foreman-proxy/dns.keytab
chown foreman-proxy:foreman-proxy /etc/foreman-proxy/dns.keytab
kinit -kt /etc/foreman-proxy/dns.keytab capsule/<CAPSULE_FQDN>@<REALM> && klist

satellite-installer --foreman-proxy-dns true \
  --foreman-proxy-dns-provider nsupdate_gss \
  --foreman-proxy-dns-managed false \
  --foreman-proxy-dns-server "<IDM_SERVER_FQDN>" \
  --foreman-proxy-dns-tsig-keytab /etc/foreman-proxy/dns.keytab \
  --foreman-proxy-dns-tsig-principal "capsule/<CAPSULE_FQDN>@<REALM>"
```

Update policy phải **nối thêm** vào giá trị cũ (`<EXISTING_POLICY>` lấy từ lệnh `dnszone-show`), không ghi đè; `\047` là ký tự `/` viết theo kiểu escape của BIND. Khi tạo keytab cho máy dự phòng cùng hostname, thêm `-r` vào `ipa-getkeytab` để không vô hiệu credential của máy đang chạy.

**Infoblox:** nhập CA của Infoblox vào `/etc/pki/ca-trust/source/anchors/` + `update-ca-trust extract`, thử `curl` tới WAPI, rồi chạy installer với `--foreman-proxy-dns-provider infoblox --enable-foreman-proxy-plugin-dns-infoblox` và các tham số `--foreman-proxy-plugin-dns-infoblox-*` (server, username, password, dns-view). Giới hạn: chỉ một DNS view, không tích hợp IPAM của Satellite vào Infoblox.

**Gắn vào Satellite:** Domain → DNS Capsule; Subnet → tab Domains chọn domain, tab Capsules chọn Reverse DNS Capsule.

**Không dùng DNS của Satellite:** bỏ Reverse DNS Capsule ở mọi subnet, bỏ DNS Capsule ở mọi domain, rồi `satellite-installer --foreman-proxy-dns false`. Host đã tạo trước đó có thể lỗi orchestration khi sửa/xoá vì thiếu bản ghi mong đợi.

## DHCP

| Provider | Dùng cho |
|---|---|
| `isc` | ISC DHCP chạy ngay trên Capsule |
| `remote_isc` | ISC DHCP ở máy khác: OMAPI để cập nhật + NFS để đọc `dhcpd.conf` và `dhcpd.leases` |
| `infoblox` | Appliance Infoblox |

```bash
# DHCP do installer quản lý (chạy bằng root)
satellite-installer --foreman-proxy-dhcp true \
  --foreman-proxy-dhcp-provider isc \
  --foreman-proxy-dhcp-managed true \
  --foreman-proxy-dhcp-range "<RANGE_START> <RANGE_END>" \
  --foreman-proxy-dhcp-gateway <GATEWAY> \
  --foreman-proxy-dhcp-nameservers <DNS_IP>
```

- Tuỳ chọn bảo vệ OMAPI: `satellite-maintain packages install bind-utils`, sinh key bằng `tsig-keygen -a hmac-sha256 <KEY_NAME>`, rồi truyền `--foreman-proxy-dhcp-key-name`, `--foreman-proxy-dhcp-key-secret` (và `--foreman-proxy-dhcp-key-algorithm` nếu khác HMAC-SHA256).
- **ISC DHCP từ xa:** trên DHCP server thêm key + `omapi-port 7911;` + `omapi-key <KEY_NAME>;` vào `dhcpd.conf`, mở 7911/tcp; tạo user/group `foreman-proxy` cùng UID/GID với Capsule; export NFS read-only `/etc/dhcp` và `/var/lib/dhcpd` chỉ cho Capsule. Trên Capsule mount vào `/srv/nfs/...` với SELinux context `dhcp_etc_t` / `dhcpd_state_t`, kiểm tra `su - foreman-proxy -c 'head -5 /srv/nfs/etc/dhcp/dhcpd.conf'`, rồi chạy installer với `--foreman-proxy-dhcp-provider remote_isc --enable-foreman-proxy-plugin-dhcp-remote-isc` và các tham số `--foreman-proxy-plugin-dhcp-remote-isc-*` (dhcp-config, dhcp-leases, key-name, key-secret, omapi-port). NFS mặc định không mã hoá.
- **Infoblox DHCP:** tương tự DNS; nếu dùng cả hai module Infoblox thì DHCP phải dùng record type `fixedaddress`.
- **Gắn vào Satellite:** Subnet → IPAM = DHCP, tab Capsules → DHCP Capsule.
- Tắt ping kiểm tra IP trống (giảm cảnh báo IDS): `satellite-installer --foreman-proxy-dhcp-ping-free-ip false`.
- **DHCP Satellite điền gì:** `next-server` = giá trị `--foreman-proxy-tftp-servername` của TFTP Capsule (nên đặt IP của Capsule), nếu trống thì lấy hostname Capsule; `filename` = boot loader theo PXE loader. PXE loader = None thì không điền hai trường này.
- IPv6-only: Satellite không quản lý DHCPv6; tự cấu hình option `bootfile-url` trỏ tới boot loader trên TFTP Capsule.

**Sự cố DHCP hay gặp:**

```bash
# Bản ghi DHCP lệch giữa Satellite và DHCP server: xem trước rồi áp dụng (perform=1)
foreman-rake orchestration:dhcp:add_missing subnet_name=<SUBNET>
foreman-rake orchestration:dhcp:add_missing subnet_name=<SUBNET> perform=1
foreman-rake orchestration:dhcp:remove_offending subnet_name=<SUBNET>
foreman-rake orchestration:dhcp:remove_offending subnet_name=<SUBNET> perform=1
```

- `remove_offending` coi mọi bản ghi không có trong Satellite là thừa — sẽ xoá cả bản ghi của bên thứ ba.
- Đổi PXE loader của host đã có gây xung đột DHCP; cách duy nhất là ghi đè bản ghi.
- Cập nhật gói `dhcpd` có thể reset quyền thư mục làm Capsule không đọc được (lỗi `Unable to set DHCP entry ... 404`).
- Tạo host xong mới đổi/bỏ DHCP Capsule của subnet → xoá host lỗi hoặc xung đột.
- `dhcpd.leases` chỉ ghi nối; bản ghi cuối cùng của cùng một lease có hiệu lực, lease bị xoá có thêm `{ deleted; }`.

## TFTP

```bash
# TFTP do installer quản lý (chạy bằng root)
satellite-installer --foreman-proxy-tftp true \
  --foreman-proxy-tftp-managed true \
  --foreman-proxy-tftp-servername <CAPSULE_IP>
```

- TFTP server có sẵn: export `/var/lib/tftpboot` qua NFS, mount vào Capsule (fstab với `rw,vers=3,nosharecache,context="system_u:object_r:tftpdir_rw_t:s0"`), rồi chạy installer với `--foreman-proxy-tftp true --foreman-proxy-tftp-root <MOUNT_POINT> --foreman-proxy-tftp-servername <TFTP_SERVER_FQDN>` và tắt chế độ managed. Tài liệu 6.19 in tham số `--foreman-proxy-managed false`, nhiều khả năng thiếu chữ `tftp`; kiểm tra bằng `satellite-installer --full-help | grep tftp-managed`.
- Gắn vào Satellite: Subnet → tab Capsules → TFTP Capsule. Tắt: bỏ TFTP Capsule ở các subnet rồi `satellite-installer --foreman-proxy-tftp false`.
- Sau khi bật/tắt tính năng bằng installer, làm mới danh sách feature của Capsule: Infrastructure > Capsules > Refresh, hoặc `hammer capsule refresh-features --name <CAPSULE_FQDN>`.

## Lỗi DNS khi tạo host

- Satellite kiểm tra trùng tên trước khi tạo bản ghi DNS, theo thứ tự: resolver hệ thống (nếu Settings "Query local nameservers" = true), nameserver của subnet, NS lấy từ SOA của domain. Timeout → kiểm tra Satellite tới được nameserver của subnet, domain có SOA, `/etc/resolv.conf` hợp lệ.
- Kinh nghiệm lab: mạng không có DNS do Satellite quản lý nhưng domain/subnet vẫn gắn DNS Capsule → tạo host báo lỗi DNS proxy 400. Cách đã làm: bỏ tick **Managed** trên interface của host. Cách triệt để: bỏ DNS Capsule khỏi domain và Reverse DNS Capsule khỏi subnet như mục "Không dùng DNS của Satellite".

## Network boot: PXE loader

| PXE loader | `filename` (x86_64) | Dùng cho |
|---|---|---|
| None | — | Không boot mạng (image-based, iPXE qua HTTP) |
| PXELinux BIOS / PXELinux UEFI | `pxelinux.0` / `pxelinux.efi` | Máy BIOS (phổ biến) / UEFI dùng PXELinux |
| Grub2 BIOS, Grub2 ELF | `grub2/grubx64.0`, `grubx64.elf` | Máy BIOS dùng GRUB2 |
| Grub2 UEFI | `grub2/grubx64.efi` | Máy UEFI qua TFTP — lựa chọn đã dùng trong lab |
| Grub2 UEFI SecureBoot | `grub2/shimx64.efi` | UEFI bật Secure Boot |
| Grub2 UEFI HTTP / HTTPS | `http://<CAPSULE_FQDN>:8000/httpboot/grub2/grubx64.efi` / `https://...:9090/...` | UEFI HTTP boot, không cần TFTP |
| Grub2 UEFI HTTPS SecureBoot | `shimx64.efi` qua HTTPS | UEFI HTTP boot + Secure Boot |
| iPXE Embedded / iPXE UEFI HTTP | URL iPXE của Satellite / `ipxe-x64.efi` | NIC có iPXE sẵn |
| iPXE Chain BIOS / UEFI | `undionly-ipxe.0` / `ipxe.efi` | PXE rồi chuyển sang iPXE qua HTTP |

Satellite tự chọn tên file theo kiến trúc (x64, aa64, ppc64...). Thứ tự boot: máy BIOS đặt network trước, đĩa sau (Satellite tự đổi cấu hình PXE về boot đĩa sau khi cài); máy UEFI đặt network trước, nhưng installer có thể đưa OS lên đầu — muốn cài lại thì dùng `efibootmgr` đổi về boot mạng. Firmware của VM phải khớp PXE loader (EFI ↔ Grub2 UEFI, BIOS ↔ PXELinux BIOS).

Bond cần cấu hình cả switch không PXE boot được; dùng boot disk để vượt qua (xem [boot disk](boot-disks-and-offline-kickstart.md)).

### UEFI, Secure Boot và cấu trúc thư mục boot loader

- Mặc định chỉ provision được Secure Boot cho cùng phiên bản RHEL với Satellite. Phiên bản RHEL khác cần shim + GRUB2 đã ký của chính phiên bản đó, đặt trên **mỗi** TFTP Capsule (cần gói `cpio`, Secure Boot chỉ có trên x86_64):

```bash
# chạy bằng root trên TFTP Capsule; tải grub2-efi-x64 và shim-x64 của RHEL đích về /tmp trước
BOOTLOADER_PATH="/var/lib/tftpboot/bootloader-universe/pxegrub2/redhat/default/x86_64"
install -o foreman-proxy -g foreman-proxy -d $BOOTLOADER_PATH
rpm2cpio /tmp/grub2-efi-x64.rpm | cpio -idv --directory /tmp
rpm2cpio /tmp/shim-x64.rpm | cpio -idv --directory /tmp
cp /tmp/boot/efi/EFI/redhat/grubx64.efi $BOOTLOADER_PATH/grubx64.efi
cp /tmp/boot/efi/EFI/redhat/shimx64.efi $BOOTLOADER_PATH/shimx64.efi
ln -sr $BOOTLOADER_PATH/grubx64.efi $BOOTLOADER_PATH/boot.efi
ln -sr $BOOTLOADER_PATH/shimx64.efi $BOOTLOADER_PATH/boot-sb.efi
chmod 644 $BOOTLOADER_PATH/grubx64.efi $BOOTLOADER_PATH/shimx64.efi
```

- Thay `default` bằng `<MAJOR>.<MINOR>` (hoặc `<MAJOR>`) chỉ khi thật sự cần binary riêng cho từng bản.
- Gốc TFTP là `/var/lib/tftpboot` (`:tftproot` trong `/etc/foreman-proxy/settings.d/tftp.yml`); HTTPBoot dùng chung gốc này (`:root_dir` trong `httpboot.yml`), URL `/httpboot/` ánh xạ thẳng vào thư mục đó.
- Khi host vào build mode, Capsule tạo `host-config/<MAC>/grub2/` chứa `grub.cfg` và symlink `boot.efi`, `boot-sb.efi`. Không có `bootloader-universe` thì dùng bố cục cũ `/var/lib/tftpboot/grub2/`.
- Satellite không cấp boot loader cho ARM — tự đặt vào `bootloader-universe`. PXELinux cần thêm `ldlinux.c32` (và tuỳ chọn `menu.c32`, `chain.c32`) do người quản trị tự cung cấp.

### UEFI HTTP boot

Truyền file qua HTTP thay TFTP, nhanh và ổn định hơn với file lớn. Cấu hình: Subnet → tab Capsules chọn Capsule cho cả **TFTP** và **HTTPBoot**; host chọn PXE loader `Grub2 UEFI HTTP`:

```bash
hammer host create --name "<HOST_NAME>" --hostgroup "<HOST_GROUP>" \
  --organization "<ORG>" --location "<LOCATION>" --mac "<MAC>" \
  --build true --enabled true --managed true --pxe-loader "Grub2 UEFI HTTP"
hammer host interface update --host "<HOST_NAME>" --managed true --primary true --provision true
```

### iPXE (deprecated, Red Hat không hỗ trợ)

Chỉ cân nhắc khi DHCP không do Satellite quản lý, firewall chặn TFTP hoặc TFTP chập chờn. Bản iPXE trong RHEL không có HTTPS. Cần bật `--foreman-proxy-httpboot true --foreman-proxy-tftp true`, cài `ipxe-bootimgs`, chép `ipxe.lkrn`/`undionly.kpxe` vào TFTP, `restorecon -RvF /var/lib/tftpboot/`, đặt `--foreman-proxy-dhcp-ipxe-bootstrap true` (Capsule), thêm Capsule vào trusted proxies, gắn template `Kickstart default iPXE`. Chi tiết: mục 8.10 (PDF tr. 60–65).

## Kiểm tra nhanh

```bash
hammer capsule info --name "<CAPSULE_FQDN>"        # Features: DNS, DHCP, TFTP, HTTPBoot...
hammer subnet info --name "<SUBNET>"              # Capsule gán cho DHCP/TFTP/DNS
hammer domain info --name "<DOMAIN>"
ls -l /var/lib/tftpboot/grub2/ /var/lib/tftpboot/host-config/   # trên TFTP Capsule
```

## Liên quan

- [Tổng quan provisioning](provisioning-overview.md) · [Boot disk và kickstart offline](boot-disks-and-offline-kickstart.md) · [Discovery](discovery.md)
- [Cổng mạng Satellite/Capsule](../architecture/network-ports.md) · [Cài Capsule](../installation/install-capsule.md)

## Nguồn

- Red Hat Satellite 6.19, Integrating provisioning infrastructure services, chương 1 Configuring DNS integration (PDF tr. 7–17) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Integrating provisioning infrastructure services, chương 2 Configuring DHCP integration (PDF tr. 18–27), chương 3 Configuring TFTP integration (PDF tr. 28–30) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Provisioning hosts, chương 2 Preparing networking for host management (PDF tr. 14–23) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Provisioning hosts, chương 8 Using network boot to provision hosts (PDF tr. 50–66), phụ lục E Boot loader management and file structure (PDF tr. 158–167) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- RH403 (Satellite 6.15), chương 8 Prepare network resources for host provisioning — import subnet từ Capsule, cú pháp installer cho dịch vụ ngoài.
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: lỗi DNS proxy 400 khi tạo host trên mạng không có DNS do Satellite quản lý, xử lý bằng bỏ tick Managed trên interface; PXE loader Grub2 UEFI cho VM EFI.
