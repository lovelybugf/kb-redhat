---
title: KVM, OpenShift Virtualization và các nhà cung cấp cloud
tags: [reference, satellite, provisioning, cloud, ocp, registration]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Provisioning hosts, chương 1, 11, 13–17 (PDF tr. 11–13, 88–95, 111–142), phụ lục B (PDF tr. 144–153); RH403 chương 9 và 12"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Ngoài VMware, Satellite 6.19 tạo được máy trên KVM (libvirt), OpenShift Virtualization (Technology Preview), Red Hat OpenStack Services on OpenShift, Amazon EC2, Google Compute Engine và Microsoft Azure. Các cloud chỉ hỗ trợ image-based provisioning, mỗi nhà cung cấp dùng kiểu xác thực riêng, và image phải được dọn sạch, cài cloud-init hoặc cho Satellite SSH vào để đăng ký host.

Trang tra cứu: mỗi mục tóm điều kiện, điểm khác biệt và trỏ tới chương tài liệu; VMware có trang riêng [compute resource VMware](compute-resource-vmware.md). Chưa có kinh nghiệm lab cho các nhà cung cấp trong trang này.

## Bảng so sánh nhanh

| Nhà cung cấp | Provider (hammer) | Xác thực | Network based | Image based | Chương (PDF tr.) |
|---|---|---|---|---|---|
| KVM (libvirt) | `Libvirt` | SSH key của user `foreman` tới hypervisor, URL `qemu+ssh` | Có | Có (qcow2 trong storage pool kiểu directory) | 11 (88–95) |
| VMware vSphere | `Vmware` | User vCenter + fingerprint | Có | Có (VMware template) | 12 (96–110) |
| OpenShift Virtualization (TP) | OpenShift Virtualization (plugin kubevirt) | Token service account | Cần Capsule quản lý mạng trên cluster | Chưa hỗ trợ provisioning từ template | 13 (111–113) |
| Red Hat OpenStack Services on OpenShift | `OpenStack` | Keystone (user, mật khẩu, project, domain) | Không | Có (image trong glance) | 14 (114–119) |
| Amazon EC2 | `EC2` | Access key + secret key | Không | Có (AMI) | 15 (120–128) |
| Google Compute Engine | `gce` | Service account key JSON | Không | Có (chỉ golden image Red Hat hỗ trợ) | 16 (129–134) |
| Microsoft Azure | `azurerm` | Client ID, client secret, subscription ID, tenant ID | Không | Có (marketplace, custom, gallery) | 17 (135–142) |

## Điểm chung khi dùng compute resource

- Tạo kết nối ở Infrastructure > Compute Resources, gán Organization/Location; mỗi nhà cung cấp có phần riêng trong compute profile (Infrastructure > Compute Profiles).
- Khai báo image (Create Image): OS, kiến trúc, user SSH, mật khẩu hoặc key, tick **User data** nếu image có cloud-init.
- Image based cấu hình sau khi boot theo một trong hai cách, và trên EC2 bật cách này thì tắt cách kia:
  - **Finish** (sau boot, qua SSH): Satellite phải SSH được vào VM bằng thông tin trong image; cần DHCP do Capsule quản lý và IP của VM khớp IP Satellite đã gán.
  - **user_data / cloud-init** (trước boot): image có cloud-init bật lúc khởi động, subnet có DHCP; template `cloud_init` không chạy sau NAT.
- Khi tạo host: Deploy on = compute resource, chọn compute profile, để trống MAC, Provisioning Method = Image Based, thêm `kt_activation_keys`.
- Mặc định xoá host không xoá VM/instance (setting "Destroy associated VM on host delete").
- Danh sách quyền tối thiểu cho người provisioning: phụ lục D; [user, role, org](../security/users-roles-orgs.md).

## KVM (libvirt)

- Chỉ quản lý được VM do Satellite tạo; chỉ hỗ trợ storage pool kiểu directory (mặc định `/var/lib/libvirt/images`).
- Network libvirt dùng để provision **không** được có DHCP riêng (network `default` của libvirt có DHCP) — Capsule phải là DHCP duy nhất.
- User không phải root trên hypervisor: `usermod -a -G libvirt <USER>`.

```bash
# Trên Satellite: key SSH cho user foreman (chạy bằng root)
su foreman -s /bin/bash -c 'ssh-keygen && ssh-copy-id root@<KVM_FQDN>'
satellite-maintain packages install libvirt-client
su foreman -s /bin/bash -c 'virsh -c qemu+ssh://root@<KVM_FQDN>/system list'

hammer compute-resource create --name "<CR_NAME>" --provider "Libvirt" \
  --url "qemu+ssh://root@<KVM_FQDN>/system" \
  --organizations "<ORG>" --locations "<LOCATION>"
hammer compute-resource image create --compute-resource "<CR_NAME>" --name "<IMAGE_NAME>" \
  --operatingsystem "<OS_TITLE>" --architecture "x86_64" --username root \
  --user-data false --uuid "/var/lib/libvirt/images/<IMAGE>.qcow2"
hammer compute-profile values create --compute-profile "<PROFILE>" --compute-resource "<CR_NAME>" \
  --compute-attributes "cpus=2,memory=4294967296" \
  --interface "compute_type=network,compute_model=virtio,compute_network=<LIBVIRT_NETWORK>" \
  --volume "pool_name=default,capacity=20G,format_type=qcow2"
```

- `memory` tính bằng byte; `--uuid` của image là đường dẫn file trên hypervisor.
- Display VNC hoặc Spice; có tuỳ chọn mật khẩu console ngẫu nhiên. Secure Boot: chọn UEFI Secure Boot ở trường Firmware của tab Virtual Machine.

## OpenShift Virtualization (Technology Preview)

- Không dùng cho production; chưa hỗ trợ provisioning từ template.
- Cần quyền get, delete, create, update, patch, list, watch với VM trong namespace (cluster role `kubevirt.io:edit`); muốn tự gợi ý network thì thêm get/list trên `network-attachment-definitions.k8s.cni.cncf.io`. Capsule quản lý mạng trên cluster, không có DHCP khác.

```bash
satellite-installer --enable-foreman-plugin-kubevirt      # trên Satellite, bằng root

oc create serviceaccount <SA_NAME> --namespace <NAMESPACE>
oc create rolebinding <SA_NAME>-kubevirt-edit --clusterrole=kubevirt.io:edit --serviceaccount=<NAMESPACE>:<SA_NAME>
oc create rolebinding <SA_NAME>-view --clusterrole=view --serviceaccount=<NAMESPACE>:<SA_NAME>
# tạo Secret kiểu kubernetes.io/service-account-token (annotation kubernetes.io/service-account.name: <SA_NAME>)
oc get secrets <SA_TOKEN_SECRET> --template='{{.data.token | base64decode}}'
```

Compute resource: Hostname của cluster, API Port, Namespace, Token, tuỳ chọn X509 CA.

## Red Hat OpenStack Services on OpenShift

- URL Keystone dạng `http://<OPENSTACK_FQDN>:5000/v3/auth/tokens`; nhập user, mật khẩu, project, user domain, project domain name/ID. Web UI ghi provider là "RHEL OpenStack Platform".
- Image phải có sẵn trong glance. Tuỳ chọn "Allow external network as main network".
- Compute profile: flavor, availability zone, image, tenant, security group, internal network, floating IP network, boot from volume + kích thước volume.

```bash
hammer compute-resource create --name "<CR_NAME>" --provider "OpenStack" \
  --url "http://<OPENSTACK_FQDN>:5000/v3/auth/tokens" \
  --user "<USER>" --password "<PASSWORD>" --tenant "<PROJECT>" \
  --domain "<USER_DOMAIN>" --project-domain-name "<PROJECT_DOMAIN>" --project-domain-id "<PROJECT_DOMAIN_ID>" \
  --organizations "<ORG>" --locations "<LOCATION>"
```

## Amazon EC2

- User AWS cần quyền `AmazonEC2FullAccess`; Satellite phải đồng bộ giờ (AWS dùng thời gian khi xác thực).
- HTTP proxy: chọn ở compute resource; proxy toàn cục trong Administer > Settings được ưu tiên hơn.
- Ngay sau khi tạo compute resource, vào tab SSH keys → Download để lưu key (sau này chỉ lấy lại được từ database: `su - postgres -c psql foreman` rồi `select secret from key_pairs where compute_resource_id = <ID>;`, lưu thành file `.pem`, `chmod 600`).
- Image: AMI ID dạng `ami-...`, user SSH, tuỳ chọn IAM role. Bật User data thì finish bị tắt và ngược lại.
- Compute profile: flavor, image, availability zone, subnet (subnet của VPC), security groups, Managed IP = Public hoặc Private.
- Finish qua SSH: Satellite tự SSH (không qua Capsule) nên phải nằm trong EC2 và đúng security group; clone `Kickstart default finish`, thêm `sudo` cho lệnh cần root, đặt tham số host `activation_keys` và `remote_execution_ssh_user` = `ec2-user`.

```bash
hammer compute-resource create --name "<CR_NAME>" --provider "EC2" \
  --user "<ACCESS_KEY_ID>" --password "<SECRET_ACCESS_KEY>" --region "<REGION>" \
  --organizations "<ORG>" --locations "<LOCATION>"
hammer compute-resource image create --compute-resource "<CR_NAME>" --name "<IMAGE_NAME>" \
  --operatingsystem "<OS_TITLE>" --architecture "x86_64" --username root \
  --user-data true --uuid "ami-<AMI_ID>"
```

## Google Compute Engine

- Chỉ dùng golden image được Red Hat hỗ trợ. Cần domain và subnet trong Satellite.
- Trong project GCE: service account có IAM role Compute, sinh key JSON; đặt metadata `enable-oslogin` = `FALSE`.
- User SSH của image không được là root, bắt đầu bằng chữ cái, chỉ chữ thường và số.
- Compute profile: machine type, image, network, tuỳ chọn ephemeral external IP (đổi khi reboot — cần IP cố định thì reserve static IP), dung lượng đĩa.
- Dùng Puppet: bật "Use UUID for certificates" (Settings > Puppet).

```bash
hammer compute-resource create --name "<CR_NAME>" --provider "gce" \
  --key-path "<KEY_FILE>.json" --zone "<ZONE>"
```

## Microsoft Azure

- Tạo Azure AD application + service principal để có client ID, tenant ID, client secret; cần subscription ID. Mỗi region một compute resource riêng.
- Hammer: `--region` viết thường, không ký tự đặc biệt; Azure Government Cloud thêm `--cloud azureusgovernment`, `azurechina` hoặc `azuregermancloud`.
- Image: user không được là root; tên image dạng `custom://<IMAGE>`, `gallery://<GALLERY>/<IMAGE>` hoặc `marketplace://<PUBLISHER>:<OFFER>:<SKU>:<VERSION>`. Image RHEL BYOS phải chấp nhận điều khoản trên Azure trước.
- Compute profile: resource group, VM size, platform Linux, username (trùng với image), mật khẩu hoặc SSH key, premium disk, caching, custom script command hoặc URI script (script phải có `sudo`; tải về `/var/lib/waagent/custom-script/download/0/`), NVIDIA driver, volume bổ sung, interface (Azure subnet, public IP, static private IP). IP private tĩnh cần subnet Satellite có Network Address trùng subnet Azure.

```bash
hammer compute-resource create --name "<CR_NAME>" --provider azurerm \
  --app-ident "<CLIENT_ID>" --secret-key "<CLIENT_SECRET>" \
  --sub-id "<SUBSCRIPTION_ID>" --tenant "<TENANT_ID>" --region "<REGION>"
```

## Kiến trúc và subscription trên cloud (RH403)

- Đặt Satellite hoặc Capsule cùng region/VPC với instance để tránh phí băng thông ra ngoài; Capsule trên cloud nối về Satellite on-premise qua VPN.
- Image **on-demand** (trả theo giờ của nhà cung cấp) cập nhật qua Red Hat Update Infrastructure, không đăng ký với RHSM hay Satellite. Image **Cloud Access (BYOS)**, kể cả gold image, có thể quản lý bằng Satellite sau khi tắt RHUI.
- Cloud không cho quản lý DHCP nên không dùng được PXE, Discovery, iPXE boot disk.

## Cloud image và đăng ký (phụ lục B)

- **Image dựng sẵn:** RHEL 9/8 KVM Guest Image (qcow2) có cloud-init; root bị khoá, user `cloud-user` có sudo; cần metadata service kiểu EC2 để nạp SSH key.
- **Image tự làm:** cài RHEL trên máy trạm có `virt-manager`, `libvirt`, `qemu-kvm`, `virt-install`, `libguestfs-tools-c`; hoặc dùng Image Builder (định dạng qcow2, vmdk, ova, vhd, ami, gce...).
- **Hoàn thiện image** (ví dụ của tài liệu cho RHEL 7, áp dụng được ý tưởng cho bản mới): cài `cloud-init` và `cloud-utils-growpart`, thêm module `resolv-conf`, đặt `NOZEROCONF=yes`, gỡ đăng ký để mọi instance không trùng danh tính, tắt máy, rồi làm sạch và nén:

```bash
subscription-manager repos --disable=*
subscription-manager unregister
# trên máy chứa image, bằng root
virt-sysprep -d <VM_NAME>
virt-sparsify --compress <VM_NAME>.qcow2 <VM_NAME>-cloud.qcow2
```

- **Chuẩn bị host để đăng ký:** chronyd đồng bộ cùng NTP với Satellite/Capsule; chép file SSL CA (đường dẫn ở Administer > Settings > Authentication > SSL CA file) vào `/etc/pki/ca-trust/source/anchors/` rồi `update-ca-trust`.
- **Đăng ký:** Hosts > Register Host, `hammer host-registration generate-command`, API `POST /api/registration_commands`, hoặc module Ansible `redhat.satellite.registration_command` — chi tiết ở [đăng ký host](../hosts/host-registration.md).

## Liên quan

- [Tổng quan provisioning](provisioning-overview.md) · [Compute resource VMware](compute-resource-vmware.md)
- [Đăng ký host](../hosts/host-registration.md) · [virt-who](../content/virt-who.md) · [Triển khai và quy hoạch](../architecture/deployment-planning.md)

## Nguồn

- Red Hat Satellite 6.19, Provisioning hosts, mục 1.4–1.5 Supported cloud providers / virtualization infrastructures (PDF tr. 13) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Provisioning hosts, chương 11 Provisioning virtual machines on KVM (PDF tr. 88–95), chương 13 OpenShift Virtualization (PDF tr. 111–113) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Provisioning hosts, chương 14 Red Hat OpenStack Services on OpenShift (PDF tr. 114–119), chương 15 Amazon EC2 (PDF tr. 120–128) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Provisioning hosts, chương 16 Google Compute Engine (PDF tr. 129–134), chương 17 Microsoft Azure (PDF tr. 135–142) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Provisioning hosts, mục 2.4 Prerequisites for image-based provisioning (PDF tr. 18), mục 5.2 Kinds of provisioning templates (PDF tr. 36–37), phụ lục B Building cloud images for Satellite (PDF tr. 144–153) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- RH403 (Satellite 6.15), chương 12 Integrate Red Hat Satellite with hybrid cloud platforms — kiến trúc trên cloud, on-demand và Cloud Access.
