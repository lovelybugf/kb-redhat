---
title: "virt-who cho subscription máy ảo: cấu hình, deploy và xử lý sự cố"
tags: [howto, satellite, virt-who, hammer]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Configuring virt-who for virtual machine subscriptions, chương 1–11 (PDF tr. 8–46); Overview, concepts, and deployment considerations, chương 4 (PDF tr. 16–17); kinh nghiệm triển khai thực tế"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** virt-who đọc danh sách hypervisor và máy ảo từ nền tảng ảo hóa rồi báo về Satellite, để subscription tính theo host (ví dụ RHEL for Virtual Datacenters) được đối soát đúng. Tạo virt-who configuration trên Satellite (Infrastructure → Virt-who Configurations hoặc `hammer virt-who-config create`), rồi deploy script lên hypervisor, Satellite, Capsule hoặc một máy RHEL riêng.

## Khi nào cần virt-who

- Máy ảo dùng subscription **theo host** (hypervisor), nhiều loại cho phép số máy ảo không giới hạn trên một hypervisor đã mua. Satellite cần biết máy ảo nào chạy trên hypervisor nào.
- Kiểm tra: **Content → Subscriptions**, cột **Requires Virt-Who** có dấu tick thì subscription đó cần virt-who.
- Với Simple Content Access, host vẫn dùng được nội dung khi chưa có virt-who; virt-who phục vụ báo cáo mức sử dụng (Subscriptions service trên Hybrid Cloud Console) và tuân thủ số lượng mua.
- Thêm cho độ chính xác: đặt system purpose cho host (qua Activation Key), bật inventory upload nếu Satellite connected, gắn máy ảo với hypervisor trên Satellite (Managing hosts, phần associating a virtual machine).

## Quy trình tổng quát

1. Import manifest có subscription theo host ([subscription manifest](subscriptions-manifest.md)).
2. Đăng ký máy sẽ chạy virt-who (hypervisor hoặc máy RHEL riêng) vào Satellite.
3. Tạo user **chỉ đọc**, mật khẩu không hết hạn trên nền tảng ảo hóa (vCenter: quyền đọc mọi đối tượng trong Data Center).
4. Tạo virt-who configuration trên Satellite.
5. Deploy configuration lên máy đích.

Mỗi configuration tự tạo user `virt_who_reporter_<N>` với role **Virt-who Reporter** (quyền tối thiểu để báo cáo); user này không đăng nhập được và không sửa tay.

## Nền tảng hỗ trợ

| Nền tảng | `--hypervisor-type` | Deploy lên | `--hypervisor-server` |
|---|---|---|---|
| VMware vSphere | `esx` | Satellite, Capsule hoặc máy RHEL riêng | FQDN/IP của vCenter |
| Nutanix AHV | `ahv` | Satellite, Capsule hoặc máy RHEL riêng | Prism Central hoặc Prism Element (chọn Prism Flavor) |
| Microsoft Hyper-V | `hyperv` | Satellite, Capsule hoặc máy RHEL riêng | FQDN/IP hypervisor |
| RHEL Virtualization (KVM) | `libvirt` | Chính hypervisor | `qemu+ssh://<USER>@<HYPERVISOR_FQDN>/system` |
| Red Hat OpenStack Services on OpenShift | `libvirt` | Chính hypervisor | `qemu:///system` |
| OpenShift Virtualization (Technology Preview) | `cnv` | Chính hypervisor | Cần file kubeconfig |

OpenShift Virtualization đang là Technology Preview, không được hỗ trợ cho production.

## Tạo configuration

Web UI: **Infrastructure → Virt-who Configurations → Create Config** → Name, Hypervisor Type, thông tin kết nối, Interval (chu kỳ báo cáo), Satellite Server FQDN, Hypervisor ID → (tùy chọn) Filtering, debug, proxy → **Submit**.

```bash
hammer virt-who-config create \
  --name "<VIRT_WHO_CONFIG>" \
  --organizations "<ORG>" \
  --interval 720 \
  --filtering-mode none \
  --hypervisor-id hostname \
  --hypervisor-type esx \
  --hypervisor-server <VCENTER_FQDN> \
  --hypervisor-username <USER> \
  --hypervisor-password <PASSWORD> \
  --satellite-url <SATELLITE_FQDN>
```

| Tùy chọn | Giá trị và lưu ý |
|---|---|
| `--interval` | Chu kỳ báo cáo, đơn vị phút |
| `--hypervisor-id` | `hostname`, `uuid` (tránh trùng khi đổi tên hypervisor), `hwuuid` (cho cấu hình trỏ vào trình quản lý ảo hóa). Đã chạy với `hwuuid` thì **không đổi sang giá trị khác** được, vì sẽ sinh bản ghi trùng |
| `--filtering-mode` | `none`, `whitelist` (chỉ lấy hypervisor liệt kê), `blacklist` (loại hypervisor liệt kê); vSphere có thêm Filter/Exclude host parents theo cluster ID. Tên host hỗ trợ wildcard và regex (escape backslash, ký tự đặc biệt đặt trong nháy) |
| `--proxy` | Ví dụ `http://<PROXY_FQDN>:3128`; để trống HTTP Proxy tương đương Ignore Proxy `*` |
| Debug | Bật Enable debugging output khi xử lý sự cố, tắt lại sau |

Giá trị **cục bộ** của từng configuration nằm ở `/etc/virt-who.d/<CONF_NAME>.conf`; giá trị **toàn cục** (Interval, debug, HTTP Proxy, Ignore Proxy) nằm ở `/etc/sysconfig/virt-who` và áp dụng cho mọi configuration trên cùng máy — deploy configuration mới sẽ ghi đè các giá trị toàn cục này.

## Deploy

- Lên Satellite Server: mở configuration → tab **Deploy** → mục **Hammer command** → copy và chạy trên Satellite (lệnh dạng `hammer virt-who-config deploy --id <ID> --organization-id <ORG_ID>`).
- Lên Capsule, hypervisor hoặc máy RHEL riêng: tab **Deploy** → **Download the script**, chép sang máy đích và chạy bằng root:

```bash
scp deploy_virt_who_config_<N>.sh root@<HOST_FQDN>:
# trên máy đích
chmod +x deploy_virt_who_config_<N>.sh
sh deploy_virt_who_config_<N>.sh
rm deploy_virt_who_config_<N>.sh
```

- Máy RHEL riêng (vật lý hoặc ảo) phải đăng ký vào Satellite và mở cổng 443 tới Satellite:

```bash
firewall-cmd --add-port="443/tcp"
firewall-cmd --add-port="443/tcp" --permanent
```

Script chứa thông tin kết nối đã mã hóa; xóa ngay sau khi chạy, không lưu vào tài liệu hay kho mã.

## Sửa, xóa configuration

```bash
hammer virt-who-config update --name "<VIRT_WHO_CONFIG>" --interval 1440
# sau mỗi lần sửa phải deploy lại lên máy đích
```

Xóa: Web UI **Actions → Delete**, rồi trên máy đích xóa `/etc/virt-who.d/<CONF_NAME>.conf`.

## Xử lý sự cố

| Triệu chứng | Cách kiểm tra / xử lý |
|---|---|
| Không biết virt-who có báo về không | Cột **Status** (OK là kết nối và báo cáo được) trong Virt-who Configurations; `hammer virt-who-config list` có thời điểm báo cáo gần nhất |
| Cần log chi tiết | Bật debug (`hammer virt-who-config update --name "<VIRT_WHO_CONFIG>" --debug true`), deploy lại, xem `/var/log/rhsm/rhsm.log`; xong thì tắt và deploy lại |
| virt-who không báo về Satellite dù cấu hình không dùng proxy | `rhsm.conf` có proxy nên Subscription Manager vẫn đi proxy; thêm `no_proxy = <SATELLITE_FQDN>` vào `/etc/rhsm/rhsm.conf` |
| Nhiều file cấu hình trên một máy, không rõ file nào lỗi | Chuyển từng file ra thư mục khác và thử lại, hoặc chạy riêng: `virt-who --debug --one-shot --config /tmp/<CONF_NAME>.conf` |
| Lỗi subscription do hypervisor hoặc máy ảo trùng | Xuất dữ liệu và đếm trùng (bên dưới); sửa hypervisor có FQDN trùng (ví dụ `localhost`) |

Chạy bằng root trên máy chạy virt-who:

```bash
systemctl stop virt-who
virt-who -op > /tmp/virt-who.json
systemctl start virt-who
# hypervisor trùng tên
json_reformat < /tmp/virt-who.json | grep name | sort | uniq -c | sort -nr | head -n5
# máy ảo trùng
json_reformat < /tmp/virt-who.json | grep guestId | sort | uniq -c | sort -nr | head -n10
# đếm số hypervisor / máy ảo đang báo
json_reformat < /tmp/virt-who.json | grep name | sort | uniq -c | wc -l
json_reformat < /tmp/virt-who.json | grep guestId | sort | uniq -c | wc -l
```

## Lưu ý từ triển khai thực tế (ẩn danh)

- Trong các triển khai đã tham chiếu, virt-who nằm ngoài phạm vi; subscription theo hypervisor được đối soát thủ công với hạ tầng ảo hóa. Nội dung trang này chủ yếu từ tài liệu hãng, chưa chạy thử.
- Khi Capsule đặt sau load balancer, virt-who kết nối thẳng tới một Satellite/Capsule cụ thể, không đi qua VIP; đưa luồng này vào bảng cổng/firewall riêng.
- Đối soát số lượng nhanh với SCA: đếm host theo OS ở Hosts → All Hosts hoặc report template trong Monitor → Reports; cột Consumed trên trang Subscriptions không phản ánh số host.

## Liên quan

- [Subscription manifest và SCA](subscriptions-manifest.md)
- [Compute resource VMware](../provisioning/compute-resource-vmware.md)
- [Cổng mạng](../architecture/network-ports.md)
- [Capsule sau load balancer](../installation/capsule-load-balancer.md)

## Nguồn

- Red Hat Satellite 6.19, Configuring virt-who for virtual machine subscriptions, chương 1 virt-who and virtual machine subscriptions (PDF tr. 8–9) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Như trên, chương 2–7 cấu hình cho RHEL Virtualization, VMware vSphere, Nutanix AHV, Microsoft Hyper-V, OpenStack Services on OpenShift, OpenShift Virtualization (PDF tr. 10–40) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Như trên, chương 8–11 sửa, xóa configuration và troubleshooting (PDF tr. 41–46) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Overview, concepts, and deployment considerations, chương 4 Expense management for Red Hat subscriptions (PDF tr. 16–17) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: virt-who ngoài phạm vi, đối soát thủ công; thiết kế luồng virt-who không qua load balancer.
