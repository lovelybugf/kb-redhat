---
title: Email thông báo, relay thư ra ngoài và webhook của Satellite
tags: [howto, satellite, monitoring, linux]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Administering Red Hat Satellite, chương 10, 19, Phụ lục F.9 (PDF tr. 58–61, 105–113, 147–148); Installing Satellite Server in a connected network environment, mục 5.8 (PDF tr. 35–36); lab (ẩn danh)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Satellite không gửi email mặc định: cấu hình SMTP ở Administer > Settings > Email, rồi bật Mail Enabled và chọn loại thông báo cho từng user. Máy chủ SMTP có chống spam (như Gmail) thì dựng Postfix relay trên Satellite; muốn đẩy sự kiện sang hệ thống khác (chat, giám sát, ITSM) thì dùng webhook.

## Cấu hình gửi email

Administer > Settings > tab **Email** (có hiệu lực ngay):

| Setting | Mặc định | Ghi chú |
|---|---|---|
| Delivery method | Sendmail | Chọn **SMTP**; `sendmail` đã deprecated |
| SMTP address / SMTP port | — / 25 | Máy chủ SMTP |
| SMTP authentication | none | `login`, `plain`... khi máy chủ cần xác thực |
| SMTP username / SMTP password | — | Tài khoản gửi thư |
| SMTP enable StartTLS auto | Yes | |
| SMTP OpenSSL verify mode | mặc định | Cách kiểm chứng chỉ khi dùng TLS |
| SMTP HELO/EHLO domain | — | |
| Email reply address, Email subject prefix | — | Địa chỉ trả lời, tiền tố tiêu đề |
| Sendmail location, Sendmail arguments | `/usr/sbin/sendmail`, `-i` | Chỉ đọc trên Web UI, sửa trong `/etc/foreman/settings.yaml` |

Ví dụ hãng đưa cho Gmail: SMTP address `smtp.gmail.com`, port `587`, authentication `plain`, StartTLS auto `Yes`, username là tài khoản Google, password là **App Password**. Hãng cũng lưu ý máy chủ SMTP có chống spam/greylisting hay gây lỗi; khi đó dựng dịch vụ SMTP relay ngay trên Satellite (mục dưới).

## Đăng ký nhận thông báo

Người nhận cần role có quyền `view_mail_notifications`. Administer > Users > chọn user:

1. Tab User: kiểm tra trường **Mail**.
2. Tab Email Preferences: bật **Mail Enabled**, chọn tần suất (daily/weekly/monthly) hoặc bật cho từng loại.
3. Submit.

Loại thông báo chính: Audit summary (lọc được bằng Mail Query), Capsule sync failure, Compliance policy summary, Content view publish/promote failure, Host built, Host errata advisory, Promote errata, Repository sync failure, Sync errata. Muốn gửi vào hộp thư nhóm: tạo một user với địa chỉ nhóm, quyền tối thiểu, đăng ký các loại cần.

Thông báo theo host (gửi cho owner của host, user hoặc user group): Hosts > All Hosts > Edit > tab Additional Information > ô **Enabled**. Tắt cho host đang lỗi đã biết để tránh ngập thư.

## Kiểm tra

- Administer > Users > user > Email Preferences > **Test email**: gửi ngay một thư thử.
- Kích hoạt toàn bộ thông báo định kỳ cho mọi user đã đăng ký (không gửi riêng một user được), chạy bằng root:

```bash
foreman-rake reports:daily
foreman-rake reports:weekly
foreman-rake reports:monthly
```

- Không nhận được: kiểm địa chỉ user, cấu hình Email, tường lửa và log máy chủ thư. Dùng Postfix cục bộ mà `mailq` báo `Mail system is down` → thư nằm trong hàng đợi, chạy `systemctl start postfix`.

## Relay thư ra ngoài qua Postfix và Gmail (lab)

Tình huống: Satellite trong lab cần gửi cảnh báo ra hộp thư Gmail. Gmail chặn thư gửi thẳng từ máy chủ không xác thực, nên Postfix trên Satellite relay qua `smtp.gmail.com:587` có xác thực SASL bằng App Password.

Phần **đã kiểm chứng** trên lab:

- Postfix cần gói SASL cho xác thực PLAIN. `dnf install` bị foreman-protector chặn trên Satellite → dùng `satellite-maintain packages install <PKG>` (thường là `cyrus-sasl-plain`; tên gói chưa ghi lại trong phiên). Lệnh này tự chạy lại `satellite-installer`, có thể restart dịch vụ.
- Satellite offline: gắn DVD/ISO RHEL đang có làm repo cục bộ để cài gói.
- Gmail phải dùng **App Password** 16 ký tự, cần bật Xác minh 2 bước trên tài khoản; dùng mật khẩu thường thì bị `535 Username and Password not accepted`. Không mở được trang App Password thường do chưa bật Xác minh 2 bước.
- Kết quả đúng: có `/etc/postfix/sasl_passwd`, `relayhost` không trống, `/var/log/maillog` có `status=sent` và Gmail trả `250 2.0.0 OK`.

Phần cấu hình Postfix dưới đây là cấu hình chuẩn của Postfix, **chưa thấy nguyên văn trong phiên kiểm chứng**; đối chiếu tài liệu RHEL trước khi dùng. Chạy bằng root:

```bash
satellite-maintain packages install cyrus-sasl-plain
postconf -e 'relayhost = [smtp.gmail.com]:587' \
  'smtp_sasl_auth_enable = yes' \
  'smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd' \
  'smtp_sasl_security_options = noanonymous' \
  'smtp_tls_security_level = encrypt'
echo '[smtp.gmail.com]:587 <USER>@gmail.com:<APP_PASSWORD>' > /etc/postfix/sasl_passwd
chmod 600 /etc/postfix/sasl_passwd
postmap /etc/postfix/sasl_passwd
systemctl enable postfix
systemctl restart postfix
```

Kiểm tra:

```bash
postconf relayhost
echo "test" | mail -s "Satellite relay test" <RECIPIENT>
grep -E 'status=|535' /var/log/maillog | tail
mailq
```

Lệnh `mail` cần gói `s-nail`; có thể thay bằng nút Test email trên Web UI. Trỏ Satellite vào relay cục bộ: Delivery method SMTP, address `localhost`, port 25, authentication none (chưa kiểm chứng), hoặc Sendmail (hãng hỗ trợ nhưng đã deprecated).

Bảo mật: App Password là bí mật; `sasl_passwd` và `sasl_passwd.db` để quyền 600, không chụp ảnh hay chép vào tài liệu (xem [Bí mật trong tài liệu](../security/documentation-secrets.md)). Thu hồi App Password khi không dùng.

## Webhook

Webhook gọi một URL bên ngoài (HTTP) khi có sự kiện trong Satellite, kiểu fire-and-forget: Satellite không chờ hoặc bỏ qua phản hồi. Sự kiện loại Actions chỉ kích hoạt khi hành động **thành công**; thất bại thì không có webhook (dùng email Repository sync failure... cho trường hợp lỗi).

### Template

Administer > Webhook > Webhook Templates > Create Template (hoặc clone mẫu có sẵn). Nội dung là ERB giống provisioning template, viết theo định dạng mà ứng dụng nhận yêu cầu. Biến dùng được: `@event_name`, `@webhook_id`, `@payload` (truy cập trường bằng `@payload[:key]`), `@payload[:context]` (request, user, org, location, IP), `@object` (đối tượng chính, không có với sự kiện custom). Danh sách hàm: `https://<SATELLITE_FQDN>/templates_doc`.

Ví dụ gửi kết quả job REX tới hệ thống chat nhận trường `text`:

```json
{
  "text": "Job <%= @object.job_invocation_id %> ket thuc voi ket qua <%= @object.task.result %>"
}
```

### Tạo webhook

Administer > Webhook > Webhooks > Create new:

1. **Subscribe to**: chọn sự kiện; **Name**; **Target URL** (có thể động); **Template**; **HTTP method**; bỏ tích Enabled nếu chưa muốn chạy.
2. Tab Credentials: User/Password nếu đích cần HTTP auth; bỏ **Verify SSL** nếu không muốn kiểm chứng chỉ đích; **Capsule Authorization** khi gọi shellhook.
3. Tab Additional: **HTTP Content Type** (`application/json`, `application/xml`, `text/plain`; Satellite không tự chuyển định dạng), HTTP headers dạng JSON (cho phép ERB).

Đích dùng cổng HTTP/HTTPS không chuẩn thì phải gán cổng đó cho SELinux trên Satellite.

Sự kiện hay dùng: Content View Promote/Publish Succeeded; Remote Execution Run Host Job Succeeded (và các biến thể errata, package, Ansible, Leapp); Build Entered/Exited; Host, Hostgroup, Subnet, Domain, User, Content View Created/Updated/Destroyed; **Status Changed** (global status của host đổi, hữu ích cho giám sát); Template Render Performed. Danh sách đầy đủ: mục 19.4.

### Shellhook trên Capsule

Webhook chỉ map một sự kiện vào một lời gọi API. Cần chạy script trên máy thì dùng plugin shellhooks của Capsule. Chạy bằng root trên Capsule:

```bash
satellite-installer --enable-foreman-proxy-plugin-shellhooks
```

- Đặt script thực thi được vào `/var/lib/foreman-proxy/shellhooks/`, tên chỉ gồm chữ, số, gạch dưới.
- Webhook gọi `https://<CAPSULE_FQDN>:9090/shellhook/<SCRIPT>` (chữ `shellhook` số ít), method POST, bật Capsule Authorization.
- Payload đi vào stdin của script; tham số qua header `X-Shellhook-Arg-1` … `X-Shellhook-Arg-99` (chỉ truyền ID, tên, label không có xuống dòng hay dấu nháy).
- stdout ghi vào log Capsule mức debug, stderr mức warning: `/var/log/foreman-proxy/proxy.log`. Lời gọi không trả giá trị.

Thử bằng curl:

```bash
curl --data "" --header "Content-Type: text/plain" \
  --header "X-Shellhook-Arg-1: Hello" --header "X-Shellhook-Arg-2: World" \
  --request POST --show-error --silent \
  https://<CAPSULE_FQDN>:9090/shellhook/<SCRIPT>
```

## Thông báo RSS trên Web UI

Biểu tượng Notifications góc phải hiển thị feed RSS (mặc định blog Red Hat Satellite, làm mới 12 giờ). Đổi hoặc tắt ở Administer > Settings > Notifications: **RSS URL**, **RSS enable** (hỗ trợ RSS 2.0 và Atom). Satellite không ra Internet thì nên tắt.

## Liên quan

- [Giám sát và chỉ số](monitoring-and-metrics.md)
- [Health check và xử lý sự cố](health-check-and-troubleshooting.md)
- [Report template và cảnh báo CVE](../hosts/report-templates-and-cve-alert.md)
- [Remote execution](../hosts/remote-execution.md)
- [Cổng mạng](../architecture/network-ports.md)
- [Bí mật trong tài liệu](../security/documentation-secrets.md)

## Nguồn

- Red Hat Satellite 6.19, Administering Red Hat Satellite, chương 10 Configuring email notifications (PDF tr. 58–61), chương 19 Using webhooks in Satellite (PDF tr. 105–113), mục 18.3 RSS (PDF tr. 102), Phụ lục F.9 Email notification settings (PDF tr. 147–148) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Installing Satellite Server in a connected network environment, mục 5.8 Configuring Satellite Server for outgoing emails (PDF tr. 35–36) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Lab Satellite 6.19 (ẩn danh), 2026: Postfix relay qua `smtp.gmail.com:587` bằng App Password, cài gói qua `satellite-maintain packages install`, kiểm tra `status=sent`/`250 2.0.0 OK`. Tham số `smtp_sasl_*`, `postmap` và cách trỏ Satellite vào relay cục bộ: chưa kiểm chứng.
