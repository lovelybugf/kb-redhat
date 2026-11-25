---
title: Chứng chỉ SSL của Satellite và Capsule (custom, gia hạn, làm mới CA)
tags: [howto, satellite, certificate, capsule, security]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Installing Satellite Server in a connected network environment, mục 5.10–5.12 (PDF tr. 40–46); Installing Capsule Server, mục 2.4–2.5 (PDF tr. 13–19); Administering Red Hat Satellite, chương 16 (PDF tr. 89–91); Managing hosts, chương 8–9 (PDF tr. 61–65); kinh nghiệm triển khai thực tế (ẩn danh)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Satellite mặc định dùng CA tự ký do installer sinh ra; muốn dùng CA doanh nghiệp thì tạo CSR đúng chuẩn và kiểm bằng `katello-certs-check` trước khi triển khai. Satellite nhận chứng chỉ qua `satellite-installer --certs-server-*`, mỗi Capsule qua `capsule-certs-generate --server-*`; đổi CA thì phải làm mới CA trên mọi host.

## Mặc định và khi nào đổi

- Installer tự tạo CA và chứng chỉ máy chủ gắn với FQDN tại thời điểm cài; CA công bố tại `http://<SATELLITE_FQDN>/pub/katello-server-ca.crt`; host nhận CA tự động khi đăng ký bằng global registration. Thư mục CA và khoá (`/root/ssl-build`) phải nằm trong phạm vi sao lưu.
- Đổi sang chứng chỉ của CA doanh nghiệp (để trình duyệt và hệ thống khác tin sẵn): làm **ngay sau khi cài Satellite, trước khi cài Capsule và đăng ký host**, tránh phải phát lại CA cho host.
- Cần thêm tên phụ (CNAME) cho Satellite: `satellite-installer --certs-cname <ALT_FQDN> --certs-update-server` (với chứng chỉ mặc định) hoặc thêm tên vào SAN của chứng chỉ custom; lên kế hoạch trước khi cài Capsule (mục 5.10).

## Yêu cầu với chứng chỉ custom (mục 5.11)

- Mã hoá PEM; khoá riêng không đặt passphrase.
- Satellite và mỗi Capsule có chứng chỉ **riêng**, nhưng do **cùng một CA** ký.
- Chứng chỉ máy chủ không phải CA (`basicConstraints = CA:FALSE`), có Key Usage `keyEncipherment`.
- CN là FQDN (không shortname) và SAN chứa đúng tên đó.
- CA trung gian: bundle phải chứa đủ chuỗi từ Root CA qua các CA trung gian.

## Tạo CSR cho Satellite

Chạy bằng root, lưu trong thư mục chỉ root đọc (không dùng `/tmp`, `/var/tmp`: hệ điều hành dọn định kỳ, installer về sau sẽ lỗi khi bật tính năng hay nâng cấp):

```bash
mkdir -p /root/satellite_cert && chmod 700 /root/satellite_cert
openssl genrsa -out /root/satellite_cert/satellite_cert_key.pem 4096
openssl pkey -noout -in /root/satellite_cert/satellite_cert_key.pem
cat > /root/satellite_cert/openssl.cnf << 'EOF'
[ req ]
req_extensions = v3_req
distinguished_name = req_distinguished_name
prompt = no

[ req_distinguished_name ]
commonName = <SATELLITE_FQDN>

[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = <SATELLITE_FQDN>
EOF
openssl req -new -key /root/satellite_cert/satellite_cert_key.pem \
  -config /root/satellite_cert/openssl.cnf \
  -out /root/satellite_cert/satellite_cert_csr.pem
openssl req -text -noout -in /root/satellite_cert/satellite_cert_csr.pem
```

Gửi CSR cho CA, ghi rõ thời hạn; nhận về chứng chỉ và CA bundle. Có thể thêm `countryName`, `organizationName`... vào phần DN.

Tự ký bằng Root CA nội bộ dựng bằng openssl (cách đã làm khi triển khai): `openssl x509 -req` **không** tự chép extension từ CSR, phải truyền `-extfile <openssl.cnf> -extensions <section>`, nếu không chứng chỉ thiếu SAN và Key Usage. Root CA chỉ có một cấp thì CA bundle chính là chứng chỉ Root CA. Khoá Root CA là tài sản nhạy cảm nhất: sao lưu offline, tốt nhất đặt CA trên máy riêng.

## Kiểm tra bộ chứng chỉ trước khi triển khai

```bash
openssl verify -CAfile /root/satellite_cert/ca_cert_bundle.pem /root/satellite_cert/satellite_cert.pem
openssl x509 -noout -pubkey -in /root/satellite_cert/satellite_cert.pem | sha256sum
openssl pkey -pubout -in /root/satellite_cert/satellite_cert_key.pem | sha256sum
katello-certs-check -t satellite \
  -b /root/satellite_cert/ca_cert_bundle.pem \
  -c /root/satellite_cert/satellite_cert.pem \
  -k /root/satellite_cert/satellite_cert_key.pem
```

Hai hash public key phải trùng. `katello-certs-check` kiểm encoding, hạn, cờ CA, SAN, khớp key và in ra đúng lệnh installer cần chạy (với `-t capsule` thì in lệnh `capsule-certs-generate`).

## Triển khai lên Satellite (mục 5.11.2)

```bash
satellite-installer --scenario satellite \
  --certs-server-cert "/root/satellite_cert/satellite_cert.pem" \
  --certs-server-key "/root/satellite_cert/satellite_cert_key.pem" \
  --certs-server-ca-cert "/root/satellite_cert/ca_cert_bundle.pem" \
  --certs-update-server --certs-update-server-ca
echo | openssl s_client -connect <SATELLITE_FQDN>:443 2>/dev/null \
  | openssl x509 -noout -subject -issuer -enddate
```

Máy trạm quản trị cần tin Root CA (Windows: Trusted Root Certification Authorities; Linux: chép vào `/etc/pki/ca-trust/source/anchors/` rồi `update-ca-trust extract`). Satellite gửi mail qua SMTP TLS do CA nội bộ ký cũng cần thêm CA vào trust store của Satellite.

## Triển khai cho Capsule (Installing Capsule Server, mục 2.4.1.2–2.4.1.3)

Tạo CSR như trên với CN/SAN là `<CAPSULE_FQDN>` (Capsule sau LB: thêm `<LB_FQDN>` vào SAN, xem [Capsule sau load balancer](capsule-load-balancer.md)). Sau khi CA ký, trên Satellite:

```bash
capsule-certs-generate \
  --foreman-proxy-fqdn <CAPSULE_FQDN> \
  --certs-tar /root/capsule_cert/<CAPSULE_FQDN>-certs.tar \
  --server-cert /root/capsule_cert/capsule_cert.pem \
  --server-key /root/capsule_cert/capsule_cert_key.pem \
  --server-ca-cert /root/capsule_cert/ca_cert_bundle.pem
```

Chép tarball sang Capsule và chạy lệnh installer được in ra (duy nhất cho từng Capsule; quy trình đầy đủ ở [Cài Capsule Server](install-capsule.md)). Giữ tarball cho các lần nâng cấp.

## Host sau khi đổi chứng chỉ

- Nếu giữ nguyên Root CA (chỉ cấp lại chứng chỉ máy chủ) thì host không phải làm gì.
- Nếu CA đổi: làm mới CA trên host (mục dưới). Gói `katello-ca-consumer-latest.noarch.rpm` (Managing hosts, mục 8.1) đã deprecated, không dùng cho triển khai mới.

## Gia hạn chứng chỉ (Administering, chương 16)

Satellite (mục 16.2): tạo CSR mới (giữ yêu cầu Key Usage/Extended Key Usage), nhận chứng chỉ mới, chạy lại **đúng lệnh** `satellite-installer --scenario satellite --certs-server-cert ... --certs-update-server --certs-update-server-ca` như lúc triển khai. Không xoá các file chứng chỉ sau khi triển khai: cần khi nâng cấp.

Capsule (mục 16.3), trên Satellite rồi trên Capsule:

```bash
capsule-certs-generate \
  --certs-tar "/root/capsule_cert/<CAPSULE_FQDN>-certs.tar" \
  --certs-update-server \
  --foreman-proxy-fqdn "<CAPSULE_FQDN>" \
  --server-ca-cert "/root/capsule_cert/ca_cert_bundle.pem" \
  --server-cert "/root/capsule_cert/capsule_cert.pem" \
  --server-key "/root/capsule_cert/capsule_cert_key.pem"
satellite-installer --scenario capsule \
  --certs-tar-file "/root/<CAPSULE_FQDN>-certs.tar" \
  --certs-update-server \
  --foreman-proxy-foreman-base-url "https://<SATELLITE_FQDN>"
```

Đặt lịch nhắc trước ngày hết hạn; kiểm tra hạn: `openssl x509 -noout -enddate -in <CERT_FILE>`.

Thay CA tự ký (mục 16.1) — dùng file CA kép tạm thời để không mất kết nối HTTPS: thêm CA mới vào file CA trên Satellite, giữ CA cũ → gia hạn chứng chỉ trên Satellite và các Capsule → phát file CA kép cho host → bỏ CA cũ khỏi file → gia hạn lần nữa → phát CA mới cho host.

## Làm mới CA trên host (Managing hosts, chương 9)

Chỉ cần khi dùng CA tự ký và CA đã đổi. Nếu đã đổi CA trên Satellite mà **không** qua file CA kép, cách chạy bằng script không nhận ra Satellite nữa → phải làm thủ công.

- REX Script: Monitor → Jobs → Run Job → category **Commands**, template **Download and run a script**, URL `https://<SATELLITE_FQDN>/unattended/public/foreman_ca_refresh` (dùng HTTP khi CA đã hết hạn).
- REX Ansible: category **Ansible Commands**, template **Download and execute a script**, cùng URL.
- Thủ công (root):

```bash
curl -o satellite_ca_cert.crt https://<SATELLITE_FQDN>/unattended/public/foreman_raw_ca
cp -u satellite_ca_cert.crt /etc/rhsm/ca/katello-server-ca.pem
cp satellite_ca_cert.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust
curl --head https://<SATELLITE_FQDN>
curl --head https://<CAPSULE_FQDN>:9090/features
```

Lệnh `curl` đầu chạy trên Satellite, rồi chép file sang host một cách an toàn (ví dụ `scp`); các lệnh còn lại chạy trên host.

## Quay về chứng chỉ mặc định

- Satellite: `satellite-installer --certs-reset`; kiểm tra trong `/etc/foreman-installer/scenarios.d/satellite-answers.yaml` các khoá `server_cert`, `server_key`, `server_cert_req`, `server_ca_cert` rỗng (mục 5.12).
- Capsule: đăng ký lại Capsule bằng global registration, rồi Infrastructure → Capsules → Refresh features (Installing Capsule Server, mục 2.5).
- Host: đăng ký lại bằng global registration (Managing hosts, mục 8.2) — xem [Đăng ký lại host](../hosts/host-reregistration.md).

## Lỗi hay gặp

| Hiện tượng | Nguyên nhân | Xử lý |
|---|---|---|
| `katello-certs-check` báo thiếu SAN/Key Usage | Ký bằng `openssl x509 -req` không kèm `-extfile/-extensions` | Ký lại có extension |
| Lỗi verify SSL giữa các thành phần | CA bundle thiếu CA trung gian; Satellite và Capsule khác CA; lệch giờ | Bổ sung chuỗi; ký lại cùng CA; `chronyc tracking` |
| Installer lỗi khi nâng cấp hoặc bật tính năng | File chứng chỉ để trong `/tmp`/`/var/tmp` đã bị dọn | Đặt lại file ở `/root/...`, chạy lại |
| Host báo lỗi chứng chỉ sau khi đổi CA | Chưa làm mới CA trên host | Mục "Làm mới CA trên host" |

## Liên quan

- [Cài Satellite connected](install-satellite-connected.md) · [Cài Capsule Server](install-capsule.md) · [Capsule sau load balancer](capsule-load-balancer.md) · [Nâng cấp và cập nhật](upgrade-satellite.md)
- [Remote execution](../hosts/remote-execution.md) · [Sao lưu và khôi phục](../operations/backup-and-restore.md) · [Bí mật trong tài liệu](../security/documentation-secrets.md)

## Nguồn

- Red Hat Satellite 6.19, Installing Satellite Server in a connected network environment, mục 5.10 Alternate CNAME, 5.11 Custom SSL certificate, 5.12 Resetting to default (PDF tr. 40–46) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Installing Capsule Server, mục 2.4 Configuring Capsule Server with SSL certificates, 2.5 Resetting (PDF tr. 13–19).
- Red Hat Satellite 6.19, Administering Red Hat Satellite, chương 16 Renewing certificates (PDF tr. 89–91).
- Red Hat Satellite 6.19, Managing hosts, chương 8 Using custom SSL certificate for hosts, chương 9 Refreshing the self-signed CA certificate on hosts (PDF tr. 61–65).
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: Root CA nội bộ ký cho Satellite và Capsule, bắt buộc `-extfile/-extensions` khi ký, kiểm tra khớp key bằng hash public key, `katello-certs-check` trước khi chạy installer, phân phối Root CA cho máy quản trị.
