---
title: Xác thực ngoài cho Satellite với LDAP, Active Directory, IdM và Keycloak
tags: [howto, satellite, authentication, idm, security, iam]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Configuring authentication for Red Hat Satellite users, chương 1–9 (PDF tr. 8–50)"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Satellite nhận năm nguồn xác thực ngoài: LDAP (chỉ mật khẩu), Active Directory nối trực tiếp và IdM (thêm Kerberos SSO), Red Hat build of Keycloak và Red Hat SSO (thêm OTP/TOTP). Người dùng ngoài nhận quyền qua user group trên Satellite có gán role và map tới nhóm bên ngoài (External groups); không map thì chỉ có role Default hoặc phải gán role bằng tay.

## Chọn nguồn xác thực

| Nguồn | Mật khẩu | SSO | OTP/TOTP | Cấu hình bằng | Chương (PDF tr.) |
|---|---|---|---|---|---|
| LDAP / LDAPS (AD qua LDAP, IdM LDAP, POSIX) | Có | Không | Không | Web UI: Administer → Authentication Sources → LDAP | 7 (43–47) |
| Active Directory nối trực tiếp (realm join) | Có | Kerberos | Không | `realm join` + `satellite-installer --foreman-ipa-authentication` | 6 (39–42) |
| Identity Management (IdM) | Có (cả user AD qua trust) | Kerberos (user Linux) | Không | `ipa-client-install` + `satellite-installer` | 3 (12–21) |
| Red Hat build of Keycloak | Có | Có (OIDC) | Có | `keycloak-httpd-client-install` + `satellite-installer --foreman-keycloak` | 5 (30–38) |
| Red Hat Single Sign-On | Có | Có | Có | Như Keycloak | 4 (22–29) — **deprecated**, RH SSO 7 hết Full Support: dùng Keycloak |

Ràng buộc quan trọng:

- Nguồn IdM và AD trực tiếp loại trừ nhau: chạy `satellite-installer --foreman-ipa-authentication*` ghi đè cấu hình nguồn còn lại.
- Một user không dùng được cả IdM lẫn LDAP: sau lần đăng nhập đầu bằng một cách, muốn đổi thì phải xoá user tự tạo trên Satellite.
- Với Keycloak/RH SSO, một client chỉ phục vụ **hoặc** Web UI **hoặc** Hammer CLI, không cả hai cùng lúc.
- Luôn giữ tài khoản `admin` nội bộ làm tài khoản khẩn cấp (break-glass). Quên mật khẩu admin: chạy bằng root trên Satellite `foreman-rake permissions:reset` (sinh mật khẩu ngẫu nhiên) hoặc `foreman-rake permissions:reset username=<USER> password=<PASSWORD>` — lưu ý lệnh này tạo admin mới nếu user chưa có và **nâng quyền admin** cho user đã có (Chương 2.4, tr. 10–11). Sau đó cập nhật `~/.hammer/cli.modules.d/foreman.yml`.

## LDAP / LDAPS

1. LDAPS: lấy CA (gồm cả CA trung gian) của máy chủ LDAP, đưa vào trust store của Satellite (chạy bằng root):

   ```bash
   cp /tmp/ldap-ca.crt /etc/pki/ca-trust/source/anchors/
   update-ca-trust extract
   rm -f /tmp/ldap-ca.crt
   satellite-maintain service restart
   ```

2. Cho SELinux mở kết nối LDAP đi ra (root): `setsebool -P nis_enabled on`.
3. Administer → Authentication Sources → LDAP → Create:
   - Tab **LDAP server**: máy chủ, cổng; chọn **LDAPS** để mã hoá.
   - Tab **Account**: tài khoản dịch vụ chỉ cần quyền bind/read/search, Base DN, Groups Base DN, LDAP filter, tuỳ chọn *Automatically Create Accounts In Satellite*, *Usergroup Sync*.
   - Tab **Attribute mappings**, **Locations**, **Organizations** (Org/Location gán cho user tự tạo) → Submit.
4. User tự tạo ở lần đăng nhập đầu **chưa có role**: quản trị viên phải gán role, tốt nhất qua external user group (mục cuối).

Quyền cần để cấu hình: `view_authenticators`, `create_authenticators`, `edit_authenticators`, `view_locations`, `assign_locations`, `view_organizations`, `assign_organizations`. Máy chủ LDAP phải theo schema RFC 2307.

Giá trị mẫu (tài khoản dịch vụ tên `<SVC_ACCOUNT>`):

| Trường | Active Directory | IdM | POSIX LDAP |
|---|---|---|---|
| Account Username | `<DOMAIN>\<SVC_ACCOUNT>` | `uid=<SVC_ACCOUNT>,cn=users,cn=accounts,dc=example,dc=com` | `uid=<SVC_ACCOUNT>,ou=users,dc=example,dc=com` |
| Base DN | `DC=example,DC=com` | `dc=example,dc=com` | `dc=example,dc=com` |
| Groups Base DN | — | `cn=groups,cn=accounts,dc=example,dc=com` | `cn=employee,ou=userclass,dc=example,dc=com` |
| Login name | `userPrincipalName` (cho phép khoảng trắng) hoặc `sAMAccountName` | `uid` | `uid` |
| First / Last / Email | `givenName` / `sn` / `mail` | `givenName` / `sn` / `mail` | `givenName` / `sn` / `mail` |

LDAP filter giới hạn ai được đăng nhập (AD):

```text
(memberOf=cn=<GROUP>,cn=Users,dc=example,dc=com)
(|(memberOf=cn=<GROUP_1>,cn=Users,dc=example,dc=com)(memberOf=cn=<GROUP_2>,cn=Users,dc=example,dc=com))
(memberOf:1.2.840.113556.1.4.1941:=cn=<NESTED_GROUP>,dc=example,dc=com)
```

Dòng cuối dùng OID matching-rule-in-chain để lấy cả thành viên của nhóm lồng nhau.

## Active Directory nối trực tiếp (Kerberos SSO)

Chạy bằng root trên Satellite:

```bash
satellite-maintain packages install adcli krb5-workstation oddjob-mkhomedir oddjob realmd \
  samba-winbind-clients samba-winbind samba-common-tools samba-winbind-krb5-locator sssd
realm join <AD_REALM> --membership-software=samba --client-software=sssd
mkdir /etc/ipa/
```

Nội dung các file cấu hình (thay `<AD_REALM>` = tên realm chữ HOA, `<AD_WORKGROUP>`, `<ad_domain>`):

```ini
# /etc/ipa/default.conf
[global]
realm = <AD_REALM>

# /etc/samba/smb.conf (phần [global])
[global]
workgroup = <AD_WORKGROUP>
realm = <AD_REALM>
kerberos method = system keytab
security = ads

# /etc/sssd/sssd.conf (thêm vào section của domain)
[domain/<ad_domain>]
ad_gpo_access_control = enforcing
ad_gpo_map_service = +foreman
```

```bash
KRB5_KTNAME=FILE:/etc/httpd/conf/http.keytab net ads keytab create HTTP -U <AD_ADMIN> -s /etc/samba/smb.conf
systemctl restart sssd
satellite-installer --foreman-ipa-authentication true
```

- `net ads keytab create` có từ Samba 4.21.1; bản cũ hơn dùng `net ads keytab add`.
- Mạng chỉ IPv6 trên RHEL ≤ 9.6 hoặc 10.0: thêm `lookup_family_order = ipv6_only` vào section domain trong `sssd.conf`.
- Kiểm tra: đăng nhập Web UI bằng UPN `<USER>@<AD_REALM>`; hoặc `kinit <USER>@<AD_REALM>` rồi `curl --negotiate --user : https://<SATELLITE_FQDN>/users/extlogin` phải chuyển hướng tới `/hosts`.
- Lỗi `Connection reset by peer - SSL_connect` khi nối AD LDAP: kiểm tra cipher bằng `openssl s_client -connect <AD_DC_FQDN>:636`; nếu là `TLS_DHE_RSA_WITH_AES_256_GCM_SHA384` thì tắt cipher đó ở phía Satellite hoặc AD.

## Identity Management (Kerberos SSO)

1. Trên IdM (quyền admin IdM): tạo host và service principal HTTP cho Satellite.

   ```bash
   kinit admin
   ipa host-add --random <SATELLITE_FQDN>
   ipa service-add HTTP/<SATELLITE_FQDN>
   ```

2. Trên Satellite (root): đưa máy vào domain bằng mật khẩu một lần vừa sinh, kiểm tra phân giải user IdM.

   ```bash
   satellite-maintain packages install ipa-client
   ipa-client-install --password '<OTP_FROM_IPA_HOST_ADD>'
   id admin
   ```

3. Bật nguồn xác thực: chỉ Web UI `satellite-installer --foreman-ipa-authentication true`; Web UI + API/Hammer thêm `--foreman-ipa-authentication-api true`. Cảnh báo của Red Hat: bật cả API thì sau khi user `kinit`, kẻ tấn công có thể lấy được phiên API mà user không cần nhập mật khẩu Satellite ở đâu.
4. (Khuyến nghị) Giới hạn ai được vào Satellite bằng HBAC trên IdM, rồi nạp lại trên Satellite:

   ```bash
   ipa hbacsvc-add satellite-prod
   ipa hbacrule-add allow-satellite-prod
   ipa hbacrule-add-service allow-satellite-prod --hbacsvcs=satellite-prod
   ipa hbacrule-add-host allow-satellite-prod --hosts=<SATELLITE_FQDN>
   ipa hbacrule-add-user allow-satellite-prod --user=<USER>
   ipa hbactest --user=<USER> --host=<SATELLITE_FQDN> --service=satellite-prod
   ipa hbacrule-disable allow_all
   ```

   Trên Satellite (root): `satellite-installer --foreman-pam-service foreman-prod`. Tắt `allow_all` ảnh hưởng mọi dịch vụ khác của IdM: làm theo bài KB "How to configure HBAC rules in IdM" trước.

5. Hammer dùng vé Kerberos: trong `~/.hammer/cli.modules.d/foreman.yml` đặt

   ```yaml
   :foreman:
     :use_sessions: true
     :default_auth_type: 'Negotiate_Auth'
   ```

   Bỏ dòng `default_auth_type` thì phải chạy `hammer auth login negotiate` sau `kinit`. Xoá vé (`kdestroy`) không đăng xuất Hammer: dùng `hammer auth logout`.

6. Trình duyệt SSO: máy dùng trình duyệt phải là client của IdM và Firefox/Chrome bật Kerberos SSO (mục 3.6–3.7). User AD qua cross-forest trust: cần external group + POSIX group trong HBAC và bổ sung thuộc tính trong SSSD trên mọi IdM server (mục 3.8, tr. 20–21).

## Red Hat build of Keycloak (SSO, TOTP)

Điều kiện: Keycloak chạy HTTPS (CA tự ký phải được tin cậy), có realm riêng cho Satellite (ví dụ `Satellite_Realm`) đã có user. Trên Satellite (root):

```bash
satellite-maintain packages install mod_auth_openidc keycloak-httpd-client-install python3-lxml
rpm --query keycloak-httpd-client-install

# client cho Web UI (dùng hammer-openidc thay cho foreman-openidc nếu muốn xác thực Hammer)
keycloak-httpd-client-install --app-name foreman-openidc \
  --keycloak-server-url "https://<KEYCLOAK_FQDN>:8443" \
  --keycloak-admin-username "admin" \
  --keycloak-realm "Satellite_Realm" \
  --keycloak-admin-realm master \
  --keycloak-auth-role root-admin \
  -t openidc -l /users/extlogin --force
satellite-installer --foreman-keycloak true \
  --foreman-keycloak-app-name "foreman-openidc" \
  --foreman-keycloak-realm "Satellite_Realm"
systemctl restart httpd
```

Phía Keycloak (realm của Satellite → Clients → client Satellite):

- Web UI: bật **Client authentication**; giữ URI `https://<SATELLITE_FQDN>/users/extlogin/redirect_uri`, thêm `https://<SATELLITE_FQDN>/users/extlogin`.
- Hammer: tắt Client authentication; giữ URI trên, thêm `urn:ietf:wg:oauth:2.0:oob`.
- Client Scopes → `<client>-dedicated` → Mappers: thêm **Audience** (Included Client Audience = client Satellite, bật *Add to ID token*) và **Group Membership** (Token Claim Name `groups`, tắt *Full group path*).

Phía Satellite (Administer → Settings → Authentication, hoặc `hammer settings set`): *Authorize login delegation* = Yes, *auth source user autocreate* = External, *Login delegation logout URL* = `https://<SATELLITE_FQDN>/users/extlogout`, OIDC Algorithm (ví dụ RS256), OIDC Audience = client ID, OIDC Issuer = `https://<KEYCLOAK_FQDN>:8443/realms/Satellite_Realm`, OIDC JWKs URL = `.../realms/Satellite_Realm/protocol/openid-connect/certs` (Keycloak khởi tạo với context path `/auth` thì chèn `/auth` trước `/realms`). Sau đó Administer → Authentication Sources → External → Edit → gán Locations/Organizations.

```bash
hammer settings set --name authorize_login_delegation --value true
hammer settings set --name login_delegation_logout_url --value https://<SATELLITE_FQDN>/users/extlogout
hammer settings set --name oidc_algorithm --value 'RS256'
hammer settings set --name oidc_audience --value "['<SATELLITE_FQDN>-hammer-openidc']"
hammer settings set --name oidc_issuer --value "https://<KEYCLOAK_FQDN>:8443/realms/Satellite_Realm"
hammer settings set --name oidc_jwks_url --value "https://<KEYCLOAK_FQDN>:8443/realms/Satellite_Realm/protocol/openid-connect/certs"
hammer auth-source external list
hammer auth-source external update --id <AUTH_SOURCE_ID> --location-ids <LOCATION_ID> --organization-ids <ORG_ID>
```

- TOTP: Keycloak → realm → Authentication → Policies → OTP Policy (FreeOTP/Google Authenticator), Required Actions → *Configure OTP* đặt *Set as default action*.
- Hammer đăng nhập: bật `:use_sessions: true`, rồi `hammer auth login oauth --oidc-token-endpoint 'https://<KEYCLOAK_FQDN>:8443/realms/Satellite_Realm/protocol/openid-connect/token' --oidc-authorization-endpoint 'https://<KEYCLOAK_FQDN>:8443' --oidc-client-id '<SATELLITE_FQDN>-hammer-openidc' --oidc-redirect-uri urn:ietf:wg:oauth:2.0:oob` (thêm `--two-factor` khi dùng TOTP, mở URL lệnh trả về để lấy success code).
- `keycloak-httpd-client-install` ≤ 1.2 yêu cầu Keycloak khởi tạo với `--http-relative-path=/auth`; từ 1.3 không bắt buộc.

## Gán quyền cho người dùng ngoài: external user groups

1. Cho ít nhất một user ngoài đăng nhập lần đầu; ghi lại tên nhóm bên ngoài (trên Satellite gắn IdM/AD: `id <USER>`).
2. Administer → User Groups → Create User Group: đặt tên (với Keycloak/RH SSO nên khác tên nhóm AD), **không** thêm user nào.
3. Tab **Roles**: chọn role (hoặc Administrator).
4. Tab **External Groups** → Add external user group: Name = tên nhóm bên ngoài **chính xác**, Auth Source = nguồn LDAP tương ứng, hoặc `EXTERNAL` cho IdM/AD trực tiếp/Keycloak → Submit.

Hành vi cần biết:

- Nhóm từ IdM/AD trực tiếp chỉ làm mới khi một thành viên đăng nhập. Nhóm LDAP làm mới theo cron mỗi 30 phút, hoặc ngay lúc đăng nhập nếu bật *Usergroup Sync* trên Auth Source; làm mới tay: `foreman-rake ldap:refresh_usergroups` hoặc User Groups → tab External Groups → Refresh.
- Không sửa được thành viên của nhóm map ngoài trên Web UI (bị ghi đè ở lần làm mới). Cần thêm quyền riêng cho một user ngoài: cho user vào một user group nội bộ không map.
- Dùng external group với LDAP thì tài khoản bind phải là anonymous hoặc tài khoản dịch vụ, không dùng biến `$login`.
- Keycloak/RH SSO không cấu hình group mapping: mọi user chỉ nhận role **Default role**.

## Gỡ Kerberos SSO

Chạy bằng root: `satellite-installer --reset-foreman-ipa-authentication` (Web UI) hoặc `--reset-foreman-ipa-authentication-api` (Web UI, API, Hammer). File SSSD đã sửa không được hoàn nguyên. Kiểm tra: `satellite-installer --help | grep foreman-ipa-authentication` phải là `false`.

## Lưu ý vận hành

- Mở tường lửa từ Satellite tới LDAP/AD (636/TCP cho LDAPS) — xem [Cổng mạng](../architecture/network-ports.md); CA của LDAP được nạp vào trust store hệ điều hành, khác với chứng chỉ của Satellite ([Chứng chỉ](../installation/certificates.md)).
- Thông điệp bắt buộc trên trang đăng nhập: Administer → Settings → General → *Login page footer text*.
- Trang này tổng hợp từ tài liệu hãng; trong các dự án đã làm, LDAP/AD chưa được bật (dùng tài khoản nội bộ), nên các bước trên chưa được kiểm chứng thực tế.

## Liên quan

- [Người dùng, role, Organization và Location](users-roles-orgs.md)
- [Chứng chỉ SSL của Satellite và Capsule](../installation/certificates.md)
- [Cổng mạng](../architecture/network-ports.md)
- [Bí mật cần che khi đưa ảnh chụp vào tài liệu](documentation-secrets.md)

## Nguồn

- Red Hat Satellite 6.19, Configuring authentication for Red Hat Satellite users — chương 1 Overview (PDF tr. 8), chương 2 Accessing Satellite web UI (tr. 9–11), chương 3 Kerberos SSO with IdM (tr. 12–21), chương 4 Red Hat Single Sign-On (tr. 22–29), chương 5 Red Hat build of Keycloak (tr. 30–38), chương 6 Kerberos SSO for AD users (tr. 39–42), chương 7 LDAP (tr. 43–47), chương 8 External user groups (tr. 48–49), chương 9 Resetting Kerberos SSO (tr. 50) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: thiết kế giữ `admin` làm tài khoản break-glass, xác thực nội bộ, LDAPS là tuỳ chọn mở 636/TCP; chưa bật LDAP/AD trong dự án.
