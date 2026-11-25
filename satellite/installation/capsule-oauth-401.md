---
title: Capsule installer lỗi 401 OAuth khi đăng ký với Satellite
tags: [troubleshooting, satellite, capsule, installation, security]
product: Red Hat Satellite 6.19
source: "Kinh nghiệm lab Satellite/Capsule 6.19.5 trên RHEL 9.8 (ẩn danh), 2026; Red Hat Satellite 6.19 — Installing Capsule Server, mục 2.4 (PDF tr. 14–15); Administering Red Hat Satellite, phụ lục F (PDF tr. 144)"
reliability: internal-tested
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Khi `satellite-installer --scenario capsule` dừng ở bước đăng ký Capsule với Satellite và báo 401, lệnh installer đang mang OAuth consumer key/secret khác cặp Foreman đang dùng, thường sau một lần rotate key. So khớp hai giá trị bằng hash, thay `--foreman-proxy-oauth-consumer-key/secret` bằng giá trị hiện hành rồi chạy lại.

## Triệu chứng

- Chạy lệnh `satellite-installer --scenario capsule ...` (do `capsule-certs-generate` in ra) trên Capsule: các bước cài gói và chứng chỉ chạy, đến bước Capsule tự đăng ký vào Foreman thì lỗi HTTP 401 (Unauthorized) liên quan OAuth, installer không báo `Success!`.
- Chạy lại cùng lệnh vẫn lỗi y hệt.
- Không nhầm với 401 của **lệnh đăng ký host** (`curl .../register` với Bearer token): đó là token hết hạn hoặc bị quoting làm hỏng, xử lý bằng cách sinh lại lệnh và truyền qua stdin (xem [Cài Capsule Server](install-capsule.md), Bước 2).

Tìm dòng lỗi (nội dung thông báo chính xác chưa được ghi lại trong lab):

```bash
grep -inE "401|unauthorized|oauth" /var/log/foreman-installer/capsule.log | tail -20
```

Trên Satellite, đối chiếu cùng thời điểm trong `/var/log/foreman/production.log`.

## Cơ chế

- Foreman (Satellite) và mọi smart proxy (Capsule, kể cả Capsule tích hợp) dùng chung **một** cặp OAuth consumer key/secret để gọi API của nhau. Cặp này giống nhau cho mọi Capsule của cùng một Satellite.
- Lệnh do `capsule-certs-generate` in ra truyền cặp đó vào `--foreman-proxy-oauth-consumer-key` và `--foreman-proxy-oauth-consumer-secret` (Installing Capsule Server, mục 2.4.1). Installer ghi các tham số này vào answers file của Capsule, nên các lần chạy sau dùng lại giá trị đã lưu.
- Trên Satellite, cặp đang hiệu lực là setting nội bộ **OAuth consumer key / OAuth consumer secret** (Administer → Settings → Authentication, hiển thị bị che; Red Hat ghi chỉ dùng nội bộ — Administering, phụ lục F, PDF tr. 144). Lab đọc giá trị này từ `/etc/foreman/settings.yaml` (khoá `:oauth_consumer_key:` và `:oauth_consumer_secret:`) và dùng thành công; tài liệu hãng không nêu tên file này.

## Nguyên nhân đã gặp

Trên lab, cặp OAuth đã được rotate (đổi trên Satellite và các Capsule cũ bằng `satellite-installer`). Sau đó `capsule-certs-generate` vẫn in ra lệnh installer mang cặp **cũ** → Capsule mới gửi chữ ký OAuth bằng key cũ → Foreman trả 401.

Vì sao `capsule-certs-generate` còn in cặp cũ chưa xác định (chưa kiểm chứng). Các nguyên nhân khác cho cùng triệu chứng, theo tài liệu triển khai: dùng lệnh/tarball sinh trên một Satellite khác; giá trị bị cắt hỏng khi copy khối lệnh từ màn hình.

## Chẩn đoán: so khớp bằng hash, không in giá trị

Giả sử lệnh installer đã được tách vào file chỉ root đọc `/root/<CAPSULE_FQDN>-install.sh` trên Satellite. Chạy bằng root trên Satellite; chỉ in 12 ký tự đầu của SHA-256:

```bash
for k in oauth_consumer_key oauth_consumer_secret; do
  printf '%s foreman=' "$k"
  sed -nE "s/^:$k:[[:space:]]*[\"']?([^\"' ]+).*/\1/p" /etc/foreman/settings.yaml \
    | tr -d '\n' | sha256sum | cut -c1-12
done
for o in key secret; do
  printf 'oauth-consumer-%s lenh=' "$o"
  grep -oE -- "--foreman-proxy-oauth-consumer-$o[[:space:]]+\"[^\"]+\"" /root/<CAPSULE_FQDN>-install.sh \
    | sed -E 's/.*"([^"]+)"$/\1/' | tr -d '\n' | sha256sum | cut -c1-12
done
```

Hai hash của key (và của secret) phải trùng nhau. Khác nhau → đúng nguyên nhân trên. Lab dùng một script Python tương đương (in 10 ký tự hash, so khớp rồi ghi đè file); khối lệnh shell ở đây viết lại theo cùng logic, chưa chạy nguyên văn.

## Cách sửa

1. Trên Satellite, thay hai giá trị trong file lệnh bằng giá trị hiện hành, không in ra màn hình (giá trị OAuth thường chỉ gồm chữ và số; nếu có ký tự đặc biệt như `|`, `&`, `\` thì sửa bằng trình soạn thảo thay cho `sed`):

```bash
KEY=$(sed -nE "s/^:oauth_consumer_key:[[:space:]]*[\"']?([^\"' ]+).*/\1/p" /etc/foreman/settings.yaml)
SECRET=$(sed -nE "s/^:oauth_consumer_secret:[[:space:]]*[\"']?([^\"' ]+).*/\1/p" /etc/foreman/settings.yaml)
sed -i -E \
  -e "s|(--foreman-proxy-oauth-consumer-key[[:space:]]+)\"[^\"]*\"|\1\"$KEY\"|" \
  -e "s|(--foreman-proxy-oauth-consumer-secret[[:space:]]+)\"[^\"]*\"|\1\"$SECRET\"|" \
  /root/<CAPSULE_FQDN>-install.sh
unset KEY SECRET
```

2. Chạy lại khối chẩn đoán: hash phải trùng.
3. Chép file sang Capsule qua stdin (không qua clipboard, không để lại bản trong `/tmp`), rồi chạy trên Capsule:

```bash
ssh root@<SATELLITE_FQDN> 'cat /root/<CAPSULE_FQDN>-install.sh' \
  | ssh root@<CAPSULE_FQDN> 'umask 077; cat > /root/<CAPSULE_FQDN>-install.sh'
ssh root@<CAPSULE_FQDN> 'bash /root/<CAPSULE_FQDN>-install.sh > /root/capsule-install.log 2>&1; grep -E "Success|running at" /root/capsule-install.log'
```

Kết quả trên lab: `Success!` và `Capsule is running at https://<CAPSULE_FQDN>:9090`.

4. Trên Satellite: `hammer capsule refresh-features --name "<CAPSULE_FQDN>"`, rồi tiếp tục gán Organization/Location/LE và sync như [Cài Capsule Server](install-capsule.md).

## Phòng ngừa

- Sau mỗi lần rotate OAuth, **không tin** khối lệnh `capsule-certs-generate` in ra: luôn so khớp hash trước khi cài Capsule mới.
- Rotate phải làm đồng bộ trong một cửa sổ bảo trì (quy trình đã chạy trên lab): sinh cặp mới vào file chỉ root đọc (ví dụ `openssl rand -hex 16` cho mỗi giá trị); trên Satellite chạy `satellite-installer` với cả `--foreman-oauth-consumer-key/secret` và `--foreman-proxy-oauth-consumer-key/secret`; trên **từng** Capsule chạy `satellite-installer --scenario capsule` với `--foreman-proxy-oauth-consumer-key/secret`; cuối cùng `hammer ping`, `hammer capsule refresh-features` và sync thử từng Capsule. Capsule bị bỏ sót sẽ mất liên lạc với Satellite.
- Lưu cặp OAuth ở két mật khẩu; không ghi vào tài liệu, không để lộ trong ảnh chụp lệnh installer ([bí mật trong tài liệu](../security/documentation-secrets.md)).
- Khi cài Capsule, tách khối lệnh installer ra file chỉ root đọc (`umask 077`) thay vì copy từ màn hình.

## Đã và chưa kiểm chứng

| Nội dung | Trạng thái |
|---|---|
| Triệu chứng 401, nguyên nhân lệch key sau rotate, sửa bằng giá trị hiện hành → Success | Đã chạy trên lab 6.19.5 / RHEL 9.8 |
| Đọc giá trị hiện hành từ `/etc/foreman/settings.yaml` | Đã dùng trên lab; tài liệu triển khai cũng gợi ý `grep oauth_consumer` file này; tài liệu hãng không nêu |
| Lý do `capsule-certs-generate` in cặp cũ | Chưa xác định |
| Khối lệnh shell chẩn đoán và sửa ở trang này | Viết lại từ script đã chạy, chưa chạy nguyên văn |

## Liên quan

- [Cài Capsule Server](install-capsule.md) · [Capsule sau load balancer](capsule-load-balancer.md) · [Chứng chỉ SSL](certificates.md)
- [Health check và xử lý sự cố](../operations/health-check-and-troubleshooting.md)

## Nguồn

- Red Hat Satellite 6.19, Installing Capsule Server, mục 2.4.1 Configuring Capsule Server with a default SSL certificate (PDF tr. 14–15) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Administering Red Hat Satellite, phụ lục F Administration settings information, Authentication settings (PDF tr. 144) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Kinh nghiệm lab (ẩn danh), 2026: Satellite/Capsule 6.19.5 trên RHEL 9.8, OAuth đã rotate; installer Capsule lỗi 401, so khớp hash và thay giá trị hiện hành thì chạy `Success`; quy trình rotate OAuth cho Satellite và các Capsule.
