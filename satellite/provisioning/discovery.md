---
title: Discovery, discovery rules và auto-provisioning
tags: [howto, satellite, provisioning, capsule, networking, hammer]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Provisioning hosts, chương 10 Discovering hosts on a network (PDF tr. 71–87), phụ lục D (PDF tr. 155–157); RH403 chương 9"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Discovery cho máy trống boot vào Discovery image (qua PXE hoặc ISO) để tự báo phần cứng về Satellite, rồi tạo host từ danh sách Hosts > Discovered Hosts thay vì nhập MAC bằng tay. Discovery rules ghép máy với host group theo điều kiện phần cứng để auto-provisioning; chế độ ISO dùng kexec là Technology Preview.

Trang này tổng hợp từ tài liệu hãng; chưa có kinh nghiệm lab riêng cho Discovery.

## Điều kiện

- Dải IP cho Discovery (khai báo trên subnet trong web UI) không được chồng lên lease pool của DHCP do installer cấu hình. Ví dụ: chia đôi một mạng, nửa đầu cho lease động, nửa sau cho reservation của host đã provision.
- Discovery Capsule phải mở được TCP **8443** tới máy đang ở Discovery image (cần cho reboot từ web UI, auto-provision, kexec). Firewall host và firewall mạng đều phải cho phép.
- Máy được discover cần ít nhất **1200 MB RAM** — image giải nén trong RAM, thiếu RAM gây kernel panic ngẫu nhiên.
- Chỉ x86_64 được hỗ trợ cho Discovery.
- Để provision từ máy đã discover: OS phải gắn template loại "Discovery kexec" (`Discovery Red Hat kexec`) và template provision (`Kickstart default`); có domain, subnet, Activation Key, kickstart content như provisioning thường.
- Quyền cho người dùng không phải admin: `view_discovered_hosts`, `submit_discovered_hosts`, `auto_provision_discovered_hosts`, `provision_discovered_hosts`, `edit_discovered_hosts`, `destroy_discovered_hosts`.

## Cài dịch vụ Discovery

Discovery bật sẵn trên Satellite; Capsule nào có TFTP đều có thể làm Discovery Capsule. Image dựa trên RHEL 9. Các lệnh chạy bằng root:

```bash
# Trên Satellite
satellite-maintain packages install foreman-discovery-image

# Trên mỗi Capsule dùng cho Discovery
satellite-installer --enable-foreman-proxy-plugin-discovery
satellite-maintain packages install foreman-discovery-image
```

- ISO nằm ở `/usr/share/foreman-discovery-image/`; image PXE được giải nén vào `/var/lib/tftpboot/boot/fdi-image/`.
- Gán Discovery Capsule cho subnet: Infrastructure > Subnets → subnet → tab Capsules → Discovery Capsule. Làm cho mọi subnet có máy cần discover.

## Chế độ PXE

Máy lạ (không có trong inventory) khi PXE boot sẽ nhận menu mặc định có hai mục: local (boot đĩa) và discovery. Mặc định là local; đổi sang discovery:

1. Administer > Settings > tab Provisioning: đặt **Default PXE global template entry** = `discovery`.
2. Hosts > Templates > Provisioning Templates → **Build PXE Default**. Satellite sinh `/var/lib/tftpboot/pxelinux.cfg/default` (BIOS) và `/var/lib/tftpboot/grub2/grub.cfg` (UEFI) rồi đẩy tới mọi TFTP Capsule.
3. Bật máy: vài phút sau màn hình trạng thái Discovery hiện, web UI báo có host mới ở Hosts > Discovered Hosts. Tên mặc định của host discover dựa trên MAC (dạng `mac<MAC_WITHOUT_COLONS>`).

Tuỳ biến tham số kernel của mục Discovery: clone snippet `pxelinux_discovery` (BIOS) hoặc `pxegrub2_discovery` (UEFI), clone template global (`PXELinux global default` / `PXEGrub2 global default`) để gọi snippet mới, chọn template đó ở setting "Global default PXE ... template", rồi Build PXE Default lại.

**Nhiều TFTP Capsule:** mọi Capsule nhận cùng một file menu, và không có macro nào render ra tên Capsule — `proxy.url` bị gán cứng. Cách xử lý: sau mỗi lần Build PXE Default, sửa URL trong `pxelinux.cfg/default` / `grub2/grub.cfg` trên từng Capsule; hoặc dùng một DNS alias chung trỏ về Capsule phù hợp theo subnet.

## Chế độ không PXE (ISO)

Dùng khi không có DHCP/TFTP do Satellite quản lý. Máy được provision bằng cách nạp lại kernel installer qua kexec, không khởi động lại — **Technology Preview**, có thể treo console hoặc lỗi hiển thị trên một số phần cứng.

1. Ghi ISO ra USB/CD (hoặc gắn vào VM):

```bash
dd bs=4M if=/usr/share/foreman-discovery-image/foreman-discovery-image-<VERSION>.iso of=/dev/<USB_DEVICE>
```

2. Boot máy từ media, chọn **Manual network setup** (nhập NIC chính, VLAN tuỳ chọn, địa chỉ, gateway, DNS) hoặc **Discovery with DHCP**.
3. Màn hình Credentials: Server URL là Satellite hoặc Discovery Capsule (Capsule thì kèm cổng, ví dụ `https://<CAPSULE_FQDN>:9090`); Connection type = Server (Satellite) hoặc Foreman Proxy (Capsule).
4. Tuỳ chọn thêm custom fact, rồi Confirm.

Đóng gói sẵn tham số để khỏi gõ tay (công cụ `discovery-remaster` có trong gói `foreman-discovery-image`):

```bash
discovery-remaster ~/iso/foreman-discovery-image-<VERSION>.iso \
  "fdi.pxip=<IP>/<PREFIX> fdi.pxgw=<GATEWAY> fdi.pxdns=<DNS_IP> \
   proxy.url=https://<CAPSULE_FQDN>:9090 proxy.type=proxy \
   fdi.pxmac=<MAC> fdi.pxauto=1"
```

## Tham số kernel hay dùng

| Tham số | Ý nghĩa |
|---|---|
| `proxy.url`, `proxy.type` | URL dịch vụ Discovery; `proxy.type=foreman` (mặc định, gửi thẳng Satellite) hoặc `proxy` khi trỏ tới Capsule (kèm cổng SSL 9090) |
| `fdi.pxip`, `fdi.pxgw`, `fdi.pxdns` | IP/prefix, gateway, DNS tĩnh cho NIC chính; bỏ trống thì dùng DHCP |
| `fdi.pxmac` | MAC của NIC chính (`AA:BB:CC:DD:EE:FF`) |
| `fdi.pxauto` | `1` tự động hoàn toàn, `0` bán tự động (hỏi xác nhận) |
| `fdi.pxfactnameN`, `fdi.pxfactvalueN` | Cặp custom fact gửi về Satellite |
| `fdi.vlan.primary` | VLAN tag cho NIC chính |
| `fdi.initnet` | `all` (mặc định) hoặc `bootif` — chỉ bật NIC đã PXE boot |
| `fdi.countdown` | Giây chờ trước khi làm mới màn hình (mặc định 45); tăng khi trạng thái báo IP là N/A |
| `fdi.dhcp_timeout`, `fdi.nmwait`, `fdi.ipwait` | Thời gian chờ DHCP (300 s), NetworkManager (120 s), IP sẵn sàng (120 s) |
| `fdi.ipv4.method`, `fdi.ipv6.method` | `ignore` để tắt stack IPv4/IPv6 (chỉ ở chế độ DHCP) |
| `fdi.ssh`, `fdi.rootpw` | Bật SSH và đặt mật khẩu root — chỉ dùng tạm khi gỡ lỗi |
| `fdi.zips`, `fdi.zipserver` | Gói mở rộng tải lúc boot và TFTP server chứa chúng |
| `net.ifnames=1`, `biosdevname=1` | Giữ cách đặt tên NIC (biosdevname cho máy Dell) để cấu hình mạng sau khi cài không lệch |

## Organization/Location tự gán cho host discover

Thứ tự Satellite áp dụng:
1. Máy thuộc một subnet đã khai báo → lấy organization/location đầu tiên của subnet.
2. Có đặt Discovery organization / Discovery location ở Administer > Settings > Discovery → dùng giá trị đó (subnet của máy cũng phải thuộc org/location này, nếu không Satellite từ chối).
3. Còn lại → org/location đầu tiên theo thứ tự tên.

Sửa hàng loạt: Hosts > Discovered Hosts → chọn máy → Select Action → Assign Organization / Assign Location.

## Tạo host từ máy đã discover

Web UI: Hosts > Discovered Hosts → **Provision** ở máy cần cài → chọn host group, organization, location → **Create Host** (hoặc **Customize Host** để chỉnh thêm). MAC được điền sẵn từ kết quả discover. Ở tab Operating System bấm **Resolve**: phải thấy `Discovery Red Hat kexec` (Discovery kexec) và `Kickstart default` (provision). Submit xong, máy tự reboot vào PXE (hoặc kexec ở chế độ ISO) và cài như provisioning thường; xong thì chuyển sang Hosts > All Hosts.

```bash
hammer discovery list
hammer discovery provision --name "<DISCOVERED_NAME>" --new-name "<HOST_NAME>" \
  --hostgroup "<HOST_GROUP>" --organization "<ORG>" --location "<LOCATION>" \
  --build true --enabled true --managed true
```

## Discovery rules và auto-provisioning

Configure > Discovery rules > Create Rule:

| Trường | Ghi chú |
|---|---|
| Search | Điều kiện theo fact, có gợi ý; ví dụ `cpu_count > 8`, kết hợp nhiều điều kiện bằng toán tử |
| Host Group | Host group dùng làm khuôn cho máy khớp rule |
| Hostname | Mẫu ERB; tên phải duy nhất, không bắt đầu bằng số, không chứa `_` hoặc `.` |
| Hosts limit | Số máy tối đa rule được cài; `0` = không giới hạn |
| Priority | Số nhỏ hơn được ưu tiên hơn |
| Enabled, Organizations, Locations | Bật rule và ngữ cảnh áp dụng |

Mẫu hostname:

```text
srv-<%= rand(99999) %>
node-<%= sequence_hostgroup_param_next("<HOST_GROUP>", 10, "discovery_host") %>
<%= @host.hostgroup.name %>-<%= @host.facts['bios_vendor'] %>-<%= rand(99999) %>
```

Nên dựa vào thông tin duy nhất từ fact (MAC, serial) để tránh trùng tên. Chạy rule: ở Discovered Hosts chọn **Auto-Provision** cho một máy, hoặc **Auto-Provision All**.

```bash
hammer discovery-rule create --name "<RULE_NAME>" --search "cpu_count > 8" \
  --hostgroup "<HOST_GROUP>" --hostname "srv-<%= rand(99999) %>" \
  --hosts-limit 5 --priority 5 --enabled true
hammer discovery auto-provision --name "<DISCOVERED_NAME>"
```

Auto-provisioning không cấu hình được NIC: host được cài với cấu hình NIC phát hiện lúc discover. Muốn khác thì chỉnh trong snippet kickstart, script hoặc Ansible sau khi cài.

## Mở rộng Discovery image

Đóng gói thêm fact, binary, driver vào file zip có cấu trúc `autostart.d/` (script chạy theo thứ tự trước khi đăng ký), `bin/` (thêm vào `PATH`), `facts/` (thêm vào `FACTERLIB`), `lib/` (thêm vào `LD_LIBRARY_PATH`, `lib/ruby` vào `RUBYLIB`). Đặt zip trên TFTP server cạnh Discovery image và thêm `fdi.zips=<ZIP1>.zip,boot/<ZIP2>.zip` (đường dẫn tính từ gốc TFTP) vào snippet Discovery. Nội dung được giải nén vào `/opt/extension`; zip sau ghi đè file trùng tên của zip trước.

## Xử lý sự cố

Máy không xuất hiện ở Hosts > Discovered Hosts nghĩa là Discovery thất bại. Kiểm tra theo thứ tự:

1. Điều kiện ở đầu trang (RAM, cổng 8443, dải IP không chồng).
2. Đã đặt `discovery` làm mặc định và đã Build PXE Default; hai file `pxelinux.cfg/default` và `grub2/grub.cfg` trên TFTP Capsule có discovery là mục mặc định.
3. `proxy.url` và `proxy.type` trong snippet `pxelinux_discovery` / `pxegrub2_discovery` đúng; nếu DNS không chắc chắn thì dùng IP trong `proxy.url`.
4. DHCP có cấp IP cho Discovery image; mạng giữa máy, Capsule và Satellite thông suốt.
5. Máy đã vào Discovery image nhưng không đăng ký được: tạm thêm `fdi.ssh=1 fdi.rootpw=<PASSWORD>` vào tham số kernel, chuyển sang TTY2 (hoặc SSH) xem log các dịch vụ `discover-host` (gửi fact lần đầu), `foreman-discovery` (làm mới fact, lệnh reboot), `nm-prepare`, `NetworkManager`. Lệnh `discovery-debug` in log, cấu hình mạng và danh sách fact — chuyển ra file rồi `scp` về để phân tích. Gỡ lỗi xong bỏ hai tham số này.

## Liên quan

- [Tổng quan provisioning](provisioning-overview.md) · [DNS, DHCP, TFTP và network boot](dns-dhcp-tftp-and-network-boot.md)
- [Boot disk và kickstart khi không có DHCP/TFTP](boot-disks-and-offline-kickstart.md)
- [Host group và tham số](../hosts/host-groups-and-parameters.md) · [User, role, org](../security/users-roles-orgs.md)

## Nguồn

- Red Hat Satellite 6.19, Provisioning hosts, chương 10 Discovering hosts on a network, mục 10.1–10.5 (PDF tr. 71–78) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Provisioning hosts, chương 10, mục 10.6–10.12 tạo host, discovery rules, mở rộng image, tham số kernel, xử lý sự cố (PDF tr. 78–87) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Provisioning hosts, phụ lục D Permissions required to provision hosts (PDF tr. 155–157) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- RH403 (Satellite 6.15), chương 9 Provision content hosts — cách giải thích Discovery PXE và không PXE.
