---
title: Quản lý cấu hình host bằng Ansible role trong Satellite
tags: [howto, satellite, ansible, remote-execution, capsule]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Managing configurations by using Ansible integration, chương 1–6 và phụ lục A–B (PDF tr. 7–22); Managing hosts, chương 6 và 19; lab Satellite 6.19 đã kiểm chứng, 2026"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Satellite dùng Ansible để giữ cấu hình host: import role từ Capsule, gán role (và ghi đè biến) cho host hoặc host group, rồi chạy Run Ansible roles. Mỗi lần chạy là một job Remote Execution qua SSH từ Capsule tới host; role của host group tự chạy khi provisioning xong, với điều kiện Capsule bật Ansible và REX, có role trên đĩa, và host nhận SSH key của Capsule.

## Mô hình hoạt động

- Job "Run Ansible roles" do Satellite giao cho một Capsule có feature **Ansible** + **Script** (REX); Capsule chạy `ansible-core` (bản 2.16 đi kèm repository Satellite) dưới user `foreman-proxy`, SSH vào host như một job REX kiểu push.
- Khác với job REX thường (lệnh chạy một lần, điều phối), role Ansible gắn vào định nghĩa của host/host group để giữ cấu hình lâu dài; chạy lại bao nhiêu lần cũng đưa host về cùng trạng thái.
- Role được đọc từ đĩa của **Capsule chạy job**, không phải từ Satellite — đây là nguyên nhân của phần lớn lỗi khi có nhiều Capsule.

Đường dẫn Satellite/Capsule import và chạy role/collection (phụ lục A):

| Đường dẫn | Dùng cho |
|---|---|
| `/etc/ansible/roles`, `/etc/ansible/collections` | Role/collection tự viết hoặc bên thứ ba |
| `/usr/share/ansible/roles`, `/usr/share/ansible/collections` | Role/collection cài từ gói RPM (ví dụ `rhel-system-roles`) |

## Điều kiện (đã kiểm chứng trên lab 6.19)

1. Capsule chạy job có feature **Ansible**, **Script** (REX), **Templates**, **Registration** (hai feature sau cần để host đăng ký/provisioning qua Capsule). Bật Ansible (root, trên từng Capsule; REX dùng `--enable-foreman-proxy-plugin-remote-execution-script`):

   ```bash
   satellite-installer --enable-foreman-proxy-plugin-ansible
   ```

   Sau đó trên Satellite: `hammer capsule refresh-features --name "<CAPSULE_FQDN>"` và kiểm tra cột Features.

2. Role có mặt ở **cùng đường dẫn trên Satellite và mọi Capsule** có thể chạy job. Red Hat khuyến nghị đồng bộ role tự viết bằng hệ thống quản lý phiên bản (Git), không chép tay từng máy.
3. Host đăng ký với **Setup REX = Yes** (global parameter `host_registration_remote_execution`, mặc định `true`; snippet `remote_execution_ssh_keys` cài key) để nhận SSH public key của Satellite và các Capsule. Host đăng ký khi tắt REX: cài key bằng tay (root trên host):

   ```bash
   mkdir -p ~/.ssh
   curl https://<CAPSULE_FQDN>:9090/ssh/pubkey >> ~/.ssh/authorized_keys
   chmod 700 ~/.ssh
   chmod 600 ~/.ssh/authorized_keys
   ```

   Hoặc từ Capsule (root): `ssh-copy-id -i ~foreman-proxy/.ssh/id_rsa_foreman_proxy.pub root@<HOST_FQDN>`.

4. Bật setting **Prefer registered through Capsule for remote execution** (Administer → Settings → Content) để job chạy qua Capsule mà host đã đăng ký; mặc định là No và Satellite chọn Capsule REX theo subnet của host, có thể rơi vào máy không SSH được tới host hoặc không có role:

   ```bash
   hammer settings set --name remote_execution_prefer_registered_through_proxy --value true
   ```

5. Mạng: Capsule → host cổng SSH (mặc định 22/TCP) — xem [Cổng mạng](../architecture/network-ports.md).

## Bước 1 — Đưa role lên và import

Ví dụ role baseline tối giản đặt ở `/etc/ansible/roles/baseline/` (trên Satellite và các Capsule):

```yaml
# defaults/main.yml
baseline_packages: [chrony, vim-enhanced, tmux]
baseline_motd: "Host duoc quan ly boi Red Hat Satellite"
```

```yaml
# tasks/main.yml
- name: Cai goi baseline
  ansible.builtin.dnf:
    name: "{{ baseline_packages }}"
    state: present
- name: Bat chronyd
  ansible.builtin.service:
    name: chronyd
    state: started
    enabled: true
- name: Ghi /etc/motd
  ansible.builtin.copy:
    dest: /etc/motd
    content: "{{ baseline_motd }}\n"
    mode: "0644"
```

Import: Configure → Ansible → Roles → **Import** → chọn Capsule chứa role → tích role → Submit. Biến trong `defaults/` được import cùng role (Configure → Ansible → Variables).

RHEL System Roles: bật AppStream rồi `satellite-maintain packages install rhel-system-roles` (role nằm ở `/usr/share/ansible/roles/`), import như trên. Role/collection cũng có thể lấy từ repository loại Ansible collection trong Satellite (Managing content ch.17).

## Bước 2 — Ghi đè biến Ansible

Configure → Ansible → Variables → chọn biến:

1. Tích **Override**, chọn Parameter Type (string, boolean, array, hash...), nhập Default Value.
2. (Tuỳ chọn) **Hidden Value** cho giá trị nhạy cảm; **Optional Input Validator** (Required, list, regex).
3. (Tuỳ chọn) **Prioritize Attribute Order** và **Specify Matchers**: giá trị khác nhau theo `hostgroup`, `os`, `domain`, `fqdn`...; ghép nhiều thuộc tính bằng dấu phẩy (AND); dòng trên cùng ưu tiên cao nhất. Kiểu array/hash có Merge Overrides, Merge Default, Avoid Duplicates.

Đặt giá trị theo phạm vi (cụ thể hơn thắng, theo thứ tự tham số của host: global → org → location → domain → subnet → OS → host group → host):

| Phạm vi | Đặt ở |
|---|---|
| Toàn cục | Configure → Global Parameters → Create Parameter (tên = tên biến, chọn Type) |
| Host group | Configure → Host Groups → tab Parameters → Add Parameter |
| Host | Hosts → All Hosts → host → tab Ansible → Variables (biểu tượng bút) |

Biến nhạy cảm trong file của role: mã hoá bằng Ansible Vault và cấu hình mật khẩu vault cho user `foreman-proxy` (chương 5, tr. 16).

## Bước 3 — Gán role cho host group hoặc host

- **Host group**: Configure → Host Groups → host group → tab **Ansible Roles** → chọn role ở Available → `+` → Submit. Host thuộc host group kế thừa role; host provisioning từ host group sẽ được áp role.
- **Host**: Hosts → All Hosts → host → Edit → tab Ansible Roles → `+` → Submit. Tab Parameters của host thêm Host Parameter nếu job template cần.
- **Thứ tự** chạy: trang host → tab Ansible Roles → kéo thả trong Assigned Ansible Roles → Submit. **Gỡ** role: Edit → Ansible Roles → `-`.

## Bước 4 — Chạy role

- Một host: Hosts → All Hosts → host → **Schedule a job → Run Ansible roles**.
- Cả host group: Configure → Host Groups → cột Actions → **Run all Ansible roles**; trang kết quả có nút Rerun.
- Theo dõi: Monitor → Jobs (output từng host, task changed/failed).
- Chạy thử không thay đổi (check mode): thêm Host Parameter `ansible_roles_check_mode`, kiểu boolean, giá trị `true`.
- Giữ trạng thái định kỳ: lên lịch job lặp lại (recurring) dùng job template chạy Ansible roles — xem [Remote Execution](../hosts/remote-execution.md).

## Role tự chạy sau provisioning (đã kiểm chứng trên lab)

Host tạo từ host group có gán role: khi provisioning xong, Satellite tự chạy các role của host group lên host mới, không cần bấm Run. Điều kiện giống mục Điều kiện ở trên; thêm: host được cài key REX trong quá trình provisioning/đăng ký và Capsule phụ trách subnet/đăng ký của host có role. Xem [Tổng quan provisioning](../provisioning/provisioning-overview.md).

## Tích hợp Ansible Automation Platform (tóm tắt)

- **Dynamic inventory**: tạo credential kiểu *Red Hat Satellite 6* (URL, user, mật khẩu) trên AAP, inventory source = Red Hat Satellite 6 (Overwrite, Overwrite Variables, Update on Launch, Cache Timeout 90). User Satellite không phải admin cần role **Ansible Tower Inventory Reader**; Satellite rất lớn thì user không phải admin làm chậm vì kiểm tra quyền (chương 6.1, tr. 17–18).
- **Provisioning callback**: thêm host parameter `ansible_tower_provisioning` = `true`, `ansible_tower_api_url` (`https://<AAP_FQDN>/api/controller/v2`), `ansible_job_template_id`, `ansible_host_config_key` vào host group; template Kickstart Default chứa snippet gọi AAP sau khi cài, dịch vụ `ansible-callback` trên host (chương 6.2, tr. 18–20).

## Hammer (chưa kiểm chứng trong lab)

Lệnh của plugin `hammer_cli_foreman_ansible`; kiểm tra cú pháp bằng `--help` trước khi đưa vào script:

```bash
hammer ansible roles list
hammer ansible roles import --proxy-id <CAPSULE_ID> --role-names baseline
hammer hostgroup ansible-roles assign --name "<HOSTGROUP>" --ansible-roles baseline
hammer host ansible-roles assign --name "<HOST_FQDN>" --ansible-roles baseline
hammer host ansible-roles play --name "<HOST_FQDN>"
hammer hostgroup ansible-roles play --name "<HOSTGROUP>"
```

## Lỗi hay gặp

| Triệu chứng | Nguyên nhân | Xử lý |
|---|---|---|
| Role không có trong danh sách Import | Role chưa nằm ở đường dẫn của Capsule đã chọn | Đặt role vào `/etc/ansible/roles` trên Capsule đó, import lại |
| Job lỗi "role not found" trên một số host | Capsule chạy job không có role (chỉ có trên Satellite) | Đồng bộ role tới mọi Capsule; bật prefer-registered |
| Job chạy từ Satellite hoặc Capsule khác zone, SSH timeout | Chọn Capsule REX theo subnet | Bật `remote_execution_prefer_registered_through_proxy`; kiểm tra REX Capsule của subnet |
| `Permission denied (publickey)` | Host không có key của Capsule chạy job | Đăng ký lại có Setup REX, hoặc cài key bằng `curl .../ssh/pubkey` |
| Thay `/etc/ansible/ansible.cfg` không có tác dụng | Có `/usr/share/foreman-proxy/.ansible.cfg` (ưu tiên hơn), hoặc biến môi trường trong `/etc/foreman-proxy/ansible.env` (do installer quản lý) ghi đè | Sửa đúng file; không sửa `ansible.env` bằng tay |

## Liên quan

- [Remote Execution](../hosts/remote-execution.md)
- [Satellite Ansible Collection](satellite-ansible-collection.md)
- [Host group và tham số](../hosts/host-groups-and-parameters.md)
- [Đăng ký host](../hosts/host-registration.md)
- [Cài Capsule](../installation/install-capsule.md)
- [Quét tuân thủ với OpenSCAP](../security/openscap-compliance.md) — role `theforeman.foreman_scap_client`

## Nguồn

- Red Hat Satellite 6.19, Managing configurations by using Ansible integration — chương 1 Configuration management with Ansible (PDF tr. 7), chương 2 Enabling the Ansible plugin (tr. 8), chương 3 Configuring Ansible definitions (tr. 9–13), chương 4 Applying Ansible definitions (tr. 14–15), chương 5 Ansible Vault (tr. 16), chương 6 Integrating with Ansible Automation Platform (tr. 17–20), phụ lục A–B (tr. 21–22) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Managing hosts — mục 6.2.1 Global parameters for registration (PDF tr. 35–36), mục 19.1.2 chọn Capsule cho REX (tr. 117), mục 19.3.8–19.3.10 phân phối SSH key (tr. 126–128), phụ lục D thứ tự tham số host (tr. 188) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Kinh nghiệm lab Satellite 6.19 (ẩn danh), 2026: import role, gán cho host và host group, Run Ansible roles thành công; role của host group tự chạy sau provisioning; các điều kiện Capsule feature, role trên Capsule chạy job, Setup REX, setting prefer-registered đã kiểm chứng.
