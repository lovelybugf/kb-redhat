---
title: Remote execution (REX) qua SSH và pull mode
tags: [howto, satellite, remote-execution, capsule, ansible, hammer]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Managing hosts, chương 19 (PDF tr. 116–145), phụ lục A–C (PDF tr. 169–187); Installing Capsule Server, mục 3.2 (PDF tr. 22); RH403 chương 7; kinh nghiệm lab (ẩn danh)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Remote execution (REX) chạy job template (Script hoặc Ansible) trên host thông qua Capsule, mặc định đẩy qua SSH bằng khoá của user `foreman-proxy`, hoặc để host tự kéo job qua MQTT (pull mode, chỉ Script). Job chạy được khi Capsule bật plugin REX, host tin khoá SSH của mọi Capsule có thể nhận job (hoặc cài pull client) và Satellite chọn đúng Capsule.

## Khái niệm cốt lõi

- **Job template**: đoạn lệnh viết bằng ERB (Script) hoặc YAML playbook có ERB (Ansible), nhận input khi chạy. Template mặc định bị khoá, muốn sửa thì Clone. Mỗi template thuộc một **Job category** và một **provider** (Script/SSH hoặc Ansible).
- **Remote Execution Features** (Administer > Remote Execution Features): ánh xạ hành động của Satellite (cài errata, cài/gỡ gói, Tracer, Leapp...) sang template cụ thể; các nút trên giao diện và `hammer job-invocation create --feature ...` đều đi qua bảng này.
- Satellite không cần kết nối trực tiếp tới host: job luôn do một Capsule (kể cả Capsule tích hợp trên Satellite) thực thi.

| | Push — chế độ `ssh` (mặc định) | Pull — chế độ `pull-mqtt` |
|---|---|---|
| Chiều kết nối | Capsule → host, TCP 22 | Host → Capsule: MQTT 1883 (nhận thông báo), HTTPS 9090 (lấy job, gửi kết quả) |
| Provider | Script và Ansible | Chỉ Script (job Ansible vẫn đi SSH) |
| Trên host cần | `sshd` + khoá công khai của Capsule trong `authorized_keys` | pull client `yggdrasil` (gói `katello-pull-transport-migrate`) |
| Thư mục chạy script | `/var/tmp` | `/run` |
| Nên dùng khi | Mặc định, mở rộng tốt hơn một chút ở quy mô lớn | Chính sách không cho mở cổng SSH vào host |

Chế độ đặt theo **từng Capsule**. Trộn chế độ giữa các Capsule thì cùng một host có thể lúc chạy pull, lúc chạy SSH — nên để mọi Capsule cùng một chế độ.

## Satellite chọn Capsule nào để chạy job

1. Lấy các interface của host có tick **Remote execution** → subnet của chúng → các Capsule REX gán cho subnet đó → chọn Capsule đang chạy ít job nhất.
2. Nếu bật **Prefer registered through Capsule for remote execution** (Administer > Settings, tab Content; mặc định No) → dùng Capsule mà host đã đăng ký qua.
3. Không tìm được và bật **Fallback to Any Capsule** → xét thêm các Capsule gán cho host ở vai trò khác (DHCP/DNS/TFTP của subnet, DNS của domain, Realm, Puppet, OpenSCAP).
4. Vẫn không có và bật **Enable Global Capsule** → bất kỳ Capsule REX nào trong Organization/Location của host.

```bash
hammer settings set --name remote_execution_prefer_registered_through_proxy --value true
hammer settings set --name remote_execution_fallback_proxy --value true
hammer settings set --name remote_execution_global_proxy --value true
```

Kinh nghiệm triển khai: khi host đăng ký qua nhiều Capsule khác nhau, bật `remote_execution_prefer_registered_through_proxy` để job đi đúng Capsule của host; nếu không, job có thể rơi vào Capsule mà host chưa tin khoá SSH. Pull mode bắt buộc bật setting này vì host chỉ nghe MQTT từ Capsule nó đăng ký.

## Chuẩn bị (checklist)

1. **Capsule có plugin**: Script (REX) và Ansible nếu chạy job Ansible. Trong lab 6.19 đã kiểm chứng, Capsule bật đủ Ansible, Script, Templates, Registration. Kiểm tra ở Infrastructure > Capsules > `<CAPSULE_FQDN>` (Active features). Chạy bằng root trên Capsule:

   ```bash
   satellite-installer --enable-foreman-proxy-plugin-remote-execution-script
   satellite-installer --enable-foreman-proxy-plugin-ansible
   ```

2. **Host đăng ký có bật REX**: Hosts > Register Host > tab Advanced, mục remote execution (hoặc `hammer host-registration generate-command ... --setup-remote-execution true`). Global parameter `host_registration_remote_execution` (mặc định `true`) điều khiển snippet `remote_execution_ssh_keys` khi đăng ký; các parameter `remote_execution_ssh_user`, `remote_execution_create_user`, `remote_execution_effective_user_method` cũng do snippet này dùng. Chi tiết: [Đăng ký host](host-registration.md).
3. **Host tin khoá SSH** của Satellite và của mọi Capsule có thể chạy job (mục dưới), `sshd` chạy, firewall mở cổng 22 từ Capsule.
4. **Quyền**: role có sẵn *Remote Execution Manager* (toàn quyền) và *Remote Execution User* (chỉ chạy job). Clone role User và lọc `execute_template_invocation` để giới hạn template/host, ví dụ `name = Reboot and host_group.name = <HOST_GROUP>`. Chạy job lên chính Satellite/Capsule cần quyền `execute_jobs_on_infrastructure_hosts`. Quyền được kiểm tra ngay trước khi job chạy, nên job hẹn giờ có thể lỗi nếu quyền bị đổi sau đó. Xem [Người dùng, role, organization](../security/users-roles-orgs.md).

## Phân phối khoá SSH (chế độ push)

Khoá công khai của Capsule nằm ở `~foreman-proxy/.ssh/id_rsa_foreman_proxy.pub`. Host cài từ Satellite (provisioning) hoặc đăng ký bằng global registration có bật REX đã tự nhận khoá. Các cách còn lại:

Chạy bằng root trên Capsule — chép khoá sang host rồi thử đăng nhập không cần mật khẩu:

```bash
ssh-copy-id -i ~foreman-proxy/.ssh/id_rsa_foreman_proxy.pub root@<HOST_FQDN>
ssh -i ~foreman-proxy/.ssh/id_rsa_foreman_proxy root@<HOST_FQDN>
```

Chạy bằng root trên host — lấy khoá qua API, lặp lại lệnh `curl` cho Satellite và cho từng Capsule:

```bash
mkdir -p ~/.ssh
curl https://<CAPSULE_FQDN>:9090/ssh/pubkey >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

- Provisioning: template Kickstart gốc đã có `<%= snippet 'remote_execution_ssh_keys' %>`; template tự viết phải thêm dòng này.
- RHEL 9 trở lên chặn root đăng nhập bằng mật khẩu, nên `ssh-copy-id` cần tạm đặt `PermitRootLogin yes` rồi trả lại mặc định sau khi chép khoá (RH403). Cách qua API hoặc qua đăng ký không vướng việc này.
- Đặt passphrase cho khoá: `ssh-keygen -p -f ~foreman-proxy/.ssh/id_rsa_foreman_proxy` trên Capsule; từ đó mỗi job phải nhập *Private key passphrase*.
- Đừng dùng tuỳ chọn *Insecure* khi sinh lệnh đăng ký nếu có triển khai khoá SSH: kẻ đứng giữa lấy được CA và JWT sẽ vào được host bằng khoá đó (Managing hosts 6.2, PDF tr. 38).
- Thay khoá bằng **SSH certificate** (CA ký) thay cho mô hình Trust On First Use: user certificate (`--foreman-proxy-plugin-remote-execution-script-ssh-user-ca-public-key-file`, host cũ phải thêm `TrustedUserCAKeys`) và host certificate (`--foreman-proxy-plugin-remote-execution-script-ssh-host-ca-public-keys-file`; bật xong host nào chưa có certificate sẽ không xác thực được). Thủ tục đầy đủ ở mục 19.4 (PDF tr. 128–134).

## Bật pull mode (MQTT)

Trên Capsule (root), rồi đặt setting ưu tiên Capsule đăng ký như mục trên:

```bash
satellite-installer --foreman-proxy-plugin-remote-execution-script-mode pull-mqtt
firewall-cmd --add-service=mqtt
firewall-cmd --runtime-to-permanent
```

Trên host: chọn **REX pull mode** ngay khi đăng ký (xem [Đăng ký host](host-registration.md)), hoặc với host đã đăng ký thì cài pull client bằng root — cần repo Satellite Client 6 và AppStream đã có trong Content View của host và đang bật:

```bash
dnf install katello-pull-transport-migrate
rpm -q yggdrasil
systemctl status yggdrasil com.redhat.Yggdrasil1.Worker1.foreman   # yggdrasil 0.4.z trở lên
systemctl status yggdrasild                                         # yggdrasil 0.2.z trở xuống
```

Quay về SSH: `satellite-installer --foreman-proxy-plugin-remote-execution-script-mode ssh`. Giới hạn số job MQTT đồng thời trên một Capsule: `--foreman-proxy-plugin-remote-execution-script-mqtt-rate-limit 200` (mặc định không giới hạn).

## Chạy job

### Web UI

Monitor > Jobs > **Run job** → chọn Job category và Job template → chọn host → nhập input → (tuỳ chọn) Advanced fields → lịch chạy → Review → **Submit**.

- Không chọn host nào thì job chạy trên **mọi host nhìn thấy trong context hiện tại** — kiểm tra kỹ trước khi Submit.
- Chọn host group không bao gồm host ở subgroup; dùng search query `hostgroup_fullname ~ "<HOST_GROUP>*"`.
- Lịch: *Immediate*, *Future* (Starts at, Starts before — quá hạn thì huỷ) hoặc *Recurring* (cron, số lần lặp, ngày kết thúc; *Purpose* là nhãn — mỗi purpose chỉ có một job active).
- Query *Static* chạy đúng danh sách host lúc tạo; *Dynamic* tính lại danh sách ngay trước khi chạy (hợp với job định kỳ).
- Cron mở rộng: `0 7 * * fri#L` (thứ Sáu cuối tháng 07:00), `0 0 * * mon#1` (thứ Hai đầu tháng), `9 0 * * sun%2` (cách một Chủ nhật).

### Hammer

```bash
hammer job-template list
hammer job-template info --id <TEMPLATE_ID>
hammer job-invocation create \
  --job-template "Run Command - Script Default" \
  --inputs command="uptime" \
  --search-query "hostgroup_fullname ~ \"<HOST_GROUP>*\""
hammer job-invocation list
hammer job-invocation output --id <JOB_ID> --host <HOST_FQDN>
hammer job-invocation cancel --id <JOB_ID>
```

Search query hay dùng: `name = <HOST_FQDN>`, `name ~ <PATTERN>`, `host_collection = <HOST_COLLECTION>`, `applicable_errata = <ERRATUM_ID>`, `lifecycle_environment = <LIFECYCLE_ENV>`. Thay `--job-template` bằng `--feature <FEATURE>` để dùng template đang gán cho feature (ví dụ `katello_errata_install`, `katello_package_install`).

### Advanced fields đáng nhớ

| Trường | Ý nghĩa |
|---|---|
| SSH user / Effective user | User kết nối và user thực thi; khác nhau thì chuyển bằng `sudo` (mặc định), `su` hoặc `dzdo` — đổi ở setting *Effective User Method*. SSH user và effective user đều không phải root thì host cần gói `acl` và filesystem thư mục làm việc hỗ trợ ACL |
| Concurrency level | Số host chạy song song; đặt 1 để vá lần lượt các node sau load balancer |
| Timeout to kill | Giây tính từ lúc bắt đầu, quá thì giết job |
| Time to pickup | Chỉ pull mode: host không nhận job trong thời gian này thì huỷ |
| Execution ordering | Theo alphabet hoặc ngẫu nhiên |

Capsule nhận job theo lô: *Allow Capsule batch tasks* (mặc định Yes) và *Capsule tasks batch size* (mặc định 100) ở Administer > Settings, tab Tasks. Job Ansible chạy theo batch nên không huỷ riêng từng host được.

## Job template tự viết

Hosts > Templates > Job templates > **New Job Template**: tab Template (tên, nội dung hoặc Import), tab Job (Job category, Provider Type *SSH* cho script hoặc *Ansible*, Timeout, Description Format như `Install package %{package_name}`), Add Input cho tham số, tab Ansible (callback gửi facts, check mode), Location/Organization. Bằng Hammer:

```bash
hammer job-template create \
  --file "<TEMPLATE_FILE>" \
  --job-category "<CATEGORY>" \
  --name "<TEMPLATE_NAME>" \
  --provider-type SSH
```

Ví dụ (theo phụ lục C, đã đổi tên gói/dịch vụ thành placeholder):

```erb
<%# Ghép template có sẵn: cài gói rồi khởi động dịch vụ %>
<%= render_template 'Package Action - Script Default', :action => 'install', :package => '<PACKAGE>' %>
<%= render_template 'Service Action - Script Default', :action => 'start', :service_name => '<SERVICE>' %>
```

```erb
<%# Template "Run Command - restorecon": category Commands, input "directory" (Required, User input) %>
restorecon -RvF <%= input("directory") %>
```

```erb
<%# Khởi động lại host: Satellite không coi việc mất SSH khi reboot là lỗi %>
<%= render_template("Power Action - Script Default", :action => "restart") %>
```

- Dùng `input("package")` trong template ghép thì phải lấy định nghĩa input từ template con: tab Job > **Add Foreign Input Set**.
- Template Ansible phải bắt đầu bằng `---`; host parameter đi vào playbook thành biến Ansible. Khi chạy trên nhiều host, Satellite chỉ dùng bản render của host đầu tiên cho cả batch — đừng rẽ nhánh theo host parameter bằng ERB, hãy dùng điều kiện Ansible.
- Kiểm thử render: nút **Preview** trong trình soạn template, chọn một host. Tra macro: Administer > About > Templates DSL, hoặc Ctrl+Space khi bật Autocompletion.
- Đổi template mặc định của một feature (Administer > Remote Execution Features) có thể làm hỏng luồng sẵn có; template clone không được Red Hat cập nhật khi nâng cấp.

## Xử lý sự cố

| Triệu chứng | Nguyên nhân thường gặp | Cách xử lý |
|---|---|---|
| `Could not resolve hostname: Name or service not known` | Host không có IP trên Satellite | Administer > Settings > Facts: *Ignore interfaces facts for provisioning* = No, đăng ký/cập nhật facts lại (mục 19.7.3) |
| Lỗi xác thực SSH (`Permission denied (publickey)`) | Job đi qua Capsule mà host chưa có khoá | Phân phối khoá của Satellite và mọi Capsule; bật ưu tiên Capsule đăng ký; xem log `journalctl -u sshd` trên host |
| Không tìm được Capsule để chạy | Subnet của host không gán Capsule REX, host không có subnet | Gán Remote Execution Capsules cho subnet hoặc bật Fallback/Global Capsule |
| Script không chạy, lỗi quyền thực thi | `/var/tmp` (push) hoặc `/run` (pull) mount `noexec` | Push: `--foreman-proxy-plugin-remote-execution-script-remote-working-dir <DIR>` (tạo thư mục, `chcon --reference=/var/tmp <DIR>`); pull: `systemctl edit` service yggdrasil, thêm `Environment=FOREMAN_YGG_WORKER_WORKDIR=<DIR>` |
| Job pull mode timeout sau khi cập nhật `yggdrasil` lên 0.4.z | Host tắt weak dependencies nên không có `foreman_ygg_migration` | `dnf install foreman_ygg_migration`, kiểm tra lại hai service yggdrasil (phụ lục A) |
| Job hẹn giờ lỗi quyền | Role của người tạo đã đổi | Quyền kiểm tra ngay trước khi chạy; tạo lại job bằng user đúng quyền |
| Job Ansible lỗi trên Capsule pull mode | Ansible chỉ đi SSH | Phân phối khoá SSH dù Capsule ở pull mode |
| Sau khi bật host certificate, host báo lỗi xác thực | Host chưa có certificate do CA ký | Ký host key, cấu hình `HostCertificate` hoặc tắt bằng `--reset-...-ssh-host-ca-public-keys-file` |

## Liên quan

- [Vá host bằng errata](patching-with-errata.md) — dùng REX để cài errata, khởi động lại.
- [Cài, gỡ, cập nhật gói và Tracer](packages-and-tracer.md)
- [Đăng ký host](host-registration.md) · [Host group và parameter](host-groups-and-parameters.md)
- [Cài Capsule](../installation/install-capsule.md) · [Cổng mạng](../architecture/network-ports.md)
- [Tích hợp Ansible](../ansible/ansible-integration.md) · [Hammer cheatsheet](../ansible/hammer-cheatsheet.md)
- [Nâng cấp RHEL bằng Leapp](rhel-major-upgrade-leapp.md) — cũng chạy qua REX.

## Nguồn

- Red Hat Satellite 6.19, Managing hosts, chương 19 Configuring and setting up remote jobs (PDF tr. 116–145); mục 6.2 tham số đăng ký (PDF tr. 36–38); phụ lục A Remote jobs timing out after yggdrasil update (PDF tr. 169); phụ lục B Template writing reference (PDF tr. 170–184); phụ lục C Job template examples (PDF tr. 185–187) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Installing Capsule Server, mục 3.2 Configuring pull-based transport (PDF tr. 22); Managing configurations by using Ansible integration, chương 2 (PDF tr. 8) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- RH403 (Satellite 6.15), chương 7 Run Remote Execution — cách giải thích, lưu ý `PermitRootLogin` trên RHEL 9.
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: lab 6.19 chạy job Script và Ansible qua Capsule thành công khi Capsule bật Ansible/Script/Templates/Registration, host đăng ký có Setup REX, nhận khoá của Satellite và các Capsule, và bật `remote_execution_prefer_registered_through_proxy`.
