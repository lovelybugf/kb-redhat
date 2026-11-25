---
title: Capsule sau load balancer
tags: [howto, satellite, capsule, networking, certificate]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Configuring Capsules with a load balancer, chương 1–10 (PDF tr. 8–37); kinh nghiệm triển khai thực tế (ẩn danh)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Đặt hai hay nhiều Capsule sau một load balancer TCP (không SSL offload) để đăng ký host và cấp nội dung vẫn chạy khi một Capsule chết. Capsule phải cài mới với `--foreman-proxy-cname`/`--certs-cname` mang FQDN của LB, LB chuyển tiếp 80/443/8000/9090, và registration/template URL trỏ về LB.

## Phạm vi và ràng buộc (chương 1–2)

- Qua LB: **đăng ký host**, **cấp nội dung** cho host, cấu hình bằng Puppet. Provisioning, virt-who, remote execution vẫn đi thẳng tới từng Capsule.
- Mọi Capsule sau LB phải có **cùng nội dung**: publish/promote CV xong thì sync tất cả; gán cùng danh sách Lifecycle Environment.
- Nâng cấp **lần lượt từng Capsule** (Capsule kém một phiên bản vẫn tương thích, nên nâng cấp cuốn chiếu không gián đoạn — xem [Nâng cấp và cập nhật](upgrade-satellite.md)).
- Không hỗ trợ chuyển Capsule đang chạy sang LB: phải cài Capsule **mới** với tham số LB ngay từ lần cài đầu.
- Phần đăng ký OS, repo, gói của Capsule giống hệt [Cài Capsule Server](install-capsule.md), Bước 1–3.

Placeholder: `<LB_FQDN>` là FQDN của VIP trên LB (có bản ghi A và PTR), `<CAPSULE_FQDN>` là FQDN từng Capsule.

## Bước 1a. Capsule dùng chứng chỉ mặc định (chương 3)

Trên Satellite, cho từng Capsule:

```bash
capsule-certs-generate \
  --foreman-proxy-fqdn <CAPSULE_FQDN> \
  --foreman-proxy-cname <LB_FQDN> \
  --certs-tar /root/capsule_cert/<CAPSULE_FQDN>-certs.tar
```

Chép tarball sang Capsule, rồi chạy lệnh installer được in ra **cộng thêm** hai tham số `--certs-cname` và `--enable-foreman-proxy-plugin-remote-execution-script`:

```bash
satellite-installer --scenario capsule \
  --certs-tar-file "/root/<CAPSULE_FQDN>-certs.tar" \
  --certs-cname "<LB_FQDN>" \
  --enable-foreman-proxy-plugin-remote-execution-script \
  --foreman-proxy-foreman-base-url "https://<SATELLITE_FQDN>" \
  --foreman-proxy-trusted-hosts "<SATELLITE_FQDN>" \
  --foreman-proxy-trusted-hosts "<CAPSULE_FQDN>" \
  --foreman-proxy-oauth-consumer-key "<OAUTH_KEY>" \
  --foreman-proxy-oauth-consumer-secret "<OAUTH_SECRET>"
```

Kiểm tra ngay trên Capsule: SAN phải có cả FQDN của Capsule lẫn của LB.

```bash
echo | openssl s_client -connect $(hostname -f):443 2>/dev/null \
  | openssl x509 -noout -subject -ext subjectAltName
```

## Bước 1b. Capsule dùng chứng chỉ của CA doanh nghiệp (chương 4)

CSR cho từng Capsule: `commonName` = FQDN của Capsule; `[alt_names]` gồm `DNS.1 = <LB_FQDN>` và `DNS.2 = <CAPSULE_FQDN>`; các yêu cầu khác như [Chứng chỉ SSL](certificates.md). Cùng một CA ký cho Satellite và mọi Capsule. Sau đó:

```bash
capsule-certs-generate \
  --foreman-proxy-fqdn <CAPSULE_FQDN> \
  --foreman-proxy-cname <LB_FQDN> \
  --certs-tar /root/capsule_cert/<CAPSULE_FQDN>-certs.tar \
  --server-ca-cert /root/capsule_cert/ca_cert_bundle.pem \
  --server-cert /root/capsule_cert/<CAPSULE_FQDN>.pem \
  --server-key /root/capsule_cert/<CAPSULE_FQDN>-key.pem
```

Installer trên Capsule bổ sung đúng hai tham số như Bước 1a. Ví dụ trong tài liệu hãng (mục 4.2) dùng cùng một tên file cho `--server-cert` và `--server-key`; thông thường đây là hai file riêng, kiểm tra lại trước khi chạy.

## Bước 1c. Khi dùng Puppet (chương 5–6, tóm tắt)

- Chọn **một** Capsule làm Puppet CA: installer thêm `--enable-puppet --puppet-server true --foreman-proxy-puppetca "true" --puppet-ca-server "<CA_CAPSULE_FQDN>" --puppet-dns-alt-names "<LB_FQDN>" --puppet-server-ca "true"`.
- Trên Capsule CA: dừng `puppetserver`, sinh chứng chỉ cho từng Capsule còn lại bằng `puppetserver ca generate --ca-client --certname <CAPSULE_FQDN> --subject-alt-names <LB_FQDN>`, bật lại `puppetserver`.
- Capsule còn lại: `satellite-maintain packages install puppetserver`, chép cert/key/CA Puppet sang `/etc/puppetlabs/puppet/ssl/`, `chown -R puppet:puppet`, `restorecon -Rv`, installer với `--foreman-proxy-puppetca "false" --puppet-server-ca "false"`.
- Giới hạn: Puppet CA không ký song song được; Capsule CA chết thì host mới không lấy được chứng chỉ Puppet. LB chuyển cổng 8141 → 8140 của Capsule CA; client dùng cổng CA 8141.

## Bước 2. Cấu hình load balancer (chương 7)

Yêu cầu bắt buộc: LB chỉ chuyển tiếp TCP, **tắt SSL offloading**, để chứng chỉ phía client đi xuyên tới Capsule (host xác thực với Capsule bằng consumer certificate).

| Dịch vụ | Cổng | Balance | Đích |
|---|---|---|---|
| HTTP | 80 | roundrobin | 80 trên mọi Capsule |
| HTTPS và RHSM | 443 | source | 443 trên mọi Capsule |
| Anaconda tải template | 8000 | roundrobin | 8000 trên mọi Capsule |
| Puppet (tuỳ chọn) | 8140 | roundrobin | 8140 trên mọi Capsule |
| PuppetCA (tuỳ chọn) | 8141 | roundrobin | 8140 chỉ trên Capsule CA |
| Capsule HTTPS: đăng ký host, OpenSCAP | 9090 | roundrobin | 9090 trên mọi Capsule |

HAProxy trên RHEL 9 (Red Hat chỉ hướng dẫn chung, LB nào hỗ trợ TCP forwarding đều được):

```bash
dnf install -y haproxy policycoreutils-python-utils
semanage boolean --modify --on haproxy_connect_any
systemctl enable --now haproxy
```

Ví dụ khung `/etc/haproxy/haproxy.cfg` cho cổng 443 (tự viết, chưa chạy thử; các cổng khác lặp lại với `balance roundrobin`; Red Hat có bài KB mẫu đầy đủ):

```text
frontend capsule-https
    bind *:443
    mode tcp
    default_backend capsule-https
backend capsule-https
    mode tcp
    balance source
    server capsule1 <CAPSULE_FQDN_1>:443 check
    server capsule2 <CAPSULE_FQDN_2>:443 check
```

LB phần cứng — các điểm đã phải chốt khi triển khai thực tế:

- Virtual server kiểu TCP thuần, **không** gắn profile SSL phía client/server; mỗi cổng một virtual server (thiếu cổng là lỗi hay gặp).
- 443 dùng persistence theo IP nguồn (đã dùng ≥ 1800 s); health monitor TCP từng cổng; idle timeout TCP ≥ 3600 s cho tải gói lớn.
- SNAT: nếu bật, Capsule thấy IP của LB thay vì IP host → firewall phải cho IP self/SNAT của LB tới Capsule; gọi `/features` qua VIP bằng chứng chỉ của Satellite có thể trả 403 (TLS vẫn thông).
- Có quy trình đặt member Disabled/Offline khi bảo trì hoặc nâng cấp từng Capsule.

## Bước 3. Đăng ký host qua load balancer (chương 9)

Trên **mọi** Capsule sau LB:

```bash
satellite-installer \
  --foreman-proxy-registration true \
  --foreman-proxy-templates true \
  --foreman-proxy-registration-url "https://<LB_FQDN>:9090" \
  --foreman-proxy-template-url "http://<LB_FQDN>:8000"
grep registration_url /etc/foreman-proxy/settings.d/registration.yml
grep template_url /etc/foreman-proxy/settings.d/templates.yml
```

Sau đó Infrastructure → Capsules → từng Capsule → Actions → **Refresh**.

Sinh lệnh đăng ký: Hosts → Register Host → chọn **một** Capsule bất kỳ trong nhóm LB (Satellite tự thay bằng URL của LB); host đã đăng ký trước đó thì bật **Force**. Bằng hammer:

```bash
hammer host-registration generate-command \
  --activation-keys "<ACTIVATION_KEY>" \
  --smart-proxy-id <CAPSULE_ID> \
  --force true
```

Trên host sau khi đăng ký, `subscription-manager config --list | grep hostname` phải ra `<LB_FQDN>`. Host đăng ký trước khi có LB: đăng ký lại với Force ([Đăng ký lại host](../hosts/host-reregistration.md)). Container: `podman login <LB_FQDN>`.

Trusted proxies trên Satellite khai báo IP **các Capsule**, không khai báo LB (LB không gọi tới Satellite) — xem [Cài Capsule Server](install-capsule.md), Bước 6.

OpenSCAP qua LB (chương 10): biến Ansible `foreman_scap_client_server` = `<LB_FQDN>` và `foreman_scap_client_port` = 9090 (override, bỏ matcher); bản Puppet tương tự. Xem [OpenSCAP](../security/openscap-compliance.md).

## Kiểm tra và thử failover (chương 8)

Tài liệu hãng: tắt OS một Capsule, chạy `subscription-manager refresh` trên host đăng ký qua LB phải vẫn thành công, rồi bật lại. Các kiểm tra bổ sung đã dùng khi triển khai (chạy từ một máy RHEL đã tin CA của Satellite, không dùng chính Capsule):

```bash
dig +short <LB_FQDN>
for p in 443 80 8000 9090; do nc -zv -w3 <LB_FQDN> $p; done
openssl s_client -connect <LB_FQDN>:443 -servername <LB_FQDN> </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -ext subjectAltName
for i in 1 2 3 4 5 6; do openssl s_client -connect <LB_FQDN>:9090 </dev/null 2>/dev/null | openssl x509 -noout -subject; done
for i in 1 2 3 4 5 6; do openssl s_client -connect <LB_FQDN>:443 </dev/null 2>/dev/null | openssl x509 -noout -subject; done
```

- Chứng chỉ trả về qua LB phải là của một Capsule, issuer là CA của Satellite, SAN có `<LB_FQDN>`. Subject là tên LB hay chứng chỉ lạ → LB đang giải mã SSL.
- 9090 (roundrobin) luân phiên các Capsule; 443 (source) luôn về cùng một Capsule.
- Không có quyền trên LB: dừng dịch vụ một Capsule bằng `satellite-maintain service stop`, chờ health monitor đánh Down, lặp lại các lệnh trên (mọi câu trả lời phải từ Capsule còn lại), trên host chạy `subscription-manager refresh` và `dnf clean all && dnf makecache`; xong thì `satellite-maintain service start` và `satellite-maintain health check`.

## Lỗi hay gặp

| Hiện tượng | Nguyên nhân | Xử lý |
|---|---|---|
| Host báo lỗi SSL khi đăng ký qua LB | Thiếu `--foreman-proxy-cname`/`--certs-cname`; LB giải mã SSL | Kiểm tra SAN qua LB; sinh lại tarball có cname; bỏ SSL profile trên LB |
| 443 thông nhưng 80/8000/9090 bị refused | LB mới tạo virtual server 443 | Bổ sung đủ cổng theo bảng Bước 2 |
| Host lúc được lúc không, lỗi tải gói | Các Capsule chưa cùng nội dung; thiếu persistence 443 | Sync mọi Capsule, so `hammer capsule content info`; bật persistence |
| Host hiện sai IP, hoặc `/features` qua VIP trả 403 | SNAT trên LB | Chấp nhận theo thiết kế, mở firewall cho IP SNAT; kiểm tra trusted proxies |

## Liên quan

- [Cài Capsule Server](install-capsule.md) · [Chứng chỉ SSL](certificates.md) · [Capsule installer lỗi 401 OAuth](capsule-oauth-401.md)
- [Lập kế hoạch triển khai](../architecture/deployment-planning.md) · [Cổng mạng](../architecture/network-ports.md) · [Đăng ký host](../hosts/host-registration.md)

## Nguồn

- Red Hat Satellite 6.19, Configuring Capsules with a load balancer: chương 1 Overview (PDF tr. 8–9), chương 2 Preparing Capsule Servers (PDF tr. 10–13), chương 3–4 default/custom SSL (PDF tr. 14–18), chương 5–6 Puppet (PDF tr. 19–28), chương 7 Load balancer (PDF tr. 29–30), chương 8 Verifying (PDF tr. 31), chương 9 Registering clients (PDF tr. 32–35), chương 10 SCAP (PDF tr. 36–37) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: cặp Capsule active–active sau LB phần cứng; yêu cầu TCP pass-through, persistence 443, idle timeout, SNAT; bộ lệnh kiểm tra SAN/luân phiên qua VIP; thử failover bằng dừng dịch vụ Capsule; các lỗi trong bảng.
