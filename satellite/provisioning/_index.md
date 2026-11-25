# Provisioning

> Cấp phát máy: template, PXE, boot disk và kickstart không cần DHCP, discovery, compute resource VMware và cloud.

## Trang

<!-- AUTO-INDEX START -->
- [Boot disk và cài kickstart khi không có DHCP/TFTP](boot-disks-and-offline-kickstart.md) — Không có DHCP/TFTP do Satellite quản lý thì cài host bằng boot disk, hoặc như cách đã thành công trong lab: boot DVD RHEL kèm ISO nhãn `OEMDRV` chứa kickstart do Satellite render.
- [KVM, OpenShift Virtualization và các nhà cung cấp cloud](cloud-and-virtualization-providers.md) — Ngoài VMware, Satellite 6.19 tạo được máy trên KVM (libvirt), OpenShift Virtualization (Technology Preview), Red Hat OpenStack Services on OpenShift, Amazon EC2, Google Compute Engine và Microsoft Azure.
- [Compute resource VMware vSphere và xử lý lỗi](compute-resource-vmware.md) — Thêm vCenter làm compute resource bằng một user riêng đủ quyền, rồi khai báo compute profile (CPU, RAM, firmware, cluster, datastore, port group) để Satellite tự tạo VM khi tạo host.
- [Discovery, discovery rules và auto-provisioning](discovery.md) — Discovery cho máy trống boot vào Discovery image (qua PXE hoặc ISO) để tự báo phần cứng về Satellite, rồi tạo host từ danh sách Hosts > Discovered Hosts thay vì nhập MAC bằng tay.
- [Tích hợp DNS, DHCP, TFTP và network boot (PXE, UEFI HTTP)](dns-dhcp-tftp-and-network-boot.md) — Capsule tự chạy DNS (BIND), DHCP (ISC), TFTP qua `satellite-installer` hoặc tích hợp dịch vụ sẵn có (DNS RFC 2136, IdM, Infoblox; ISC DHCP từ xa; TFTP qua NFS), rồi gán vào domain, subnet.
- [Tổng quan provisioning trong Satellite](provisioning-overview.md) — Provisioning biến một máy trống thành host RHEL đã cài, đã đăng ký và đã cấu hình, dựa trên bốn khối: nội dung cài đặt, template, tài nguyên mạng và host group.
<!-- AUTO-INDEX END -->
