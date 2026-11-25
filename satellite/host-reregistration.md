---
title: Huỷ đăng ký và đăng ký lại host lên Satellite (global registration)
tags: [runbook, satellite, registration, hammer]
product: Red Hat Satellite 6.15–6.19, RHEL 9
source: "Satellite 6.19 Managing hosts, Registering hosts to Satellite; Phiên Claude Code 2026-11-25 (dc97f4ce)"
reliability: internal-tested
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** `subscription-manager unregister` và `clean` không đụng tới `/etc/rhsm/rhsm.conf`, nên host vẫn trỏ về Satellite cũ; đừng xoá `rhsm.conf` mà để lệnh global registration tự cấu hình lại, chỉ cần dọn CA cũ và proxy cũ. Muốn bản ghi host mới trên Satellite thì xoá host bằng hammer trước; muốn ghi đè đăng ký đang có thì bật Force.

## Khi nào dùng

Chuyển host sang Satellite khác, đổi Organization, hoặc đăng ký lại host lỗi chứng chỉ. Chỉ đổi lifecycle environment, content view hay Capsule của host đang chạy thì không cần đăng ký lại — xem [Activation Key chỉ áp dụng lúc đăng ký](activation-key-registration-only.md).

## Lệnh global registration tự làm gì

Lệnh sinh ra từ **Hosts → Register Host** (`curl ... | bash`) tự:

- sao lưu `/etc/rhsm/rhsm.conf` thành `rhsm.conf.bak` (chỉ lần đầu);
- chép CA của Satellite vào `/etc/rhsm/ca/katello-server-ca.pem`;
- chạy `subscription-manager config` (hostname, port, prefix, baseurl, repo_ca_cert);
- chạy `subscription-manager register` với `<ORG>` và `<ACTIVATION_KEY>`.

Vì vậy không phải sửa tay `rhsm.conf`. Cách cũ dùng gói `katello-ca-consumer-*` đã deprecated (Managing hosts 6.19, mục 6.5).

## Các bước

### 1. Trên host: huỷ đăng ký và dọn cấu hình cũ (root)

```bash
subscription-manager unregister
subscription-manager clean
rm -f /etc/rhsm/ca/katello-*.pem
```

- **Không** xoá `/etc/rhsm/rhsm.conf`: script đăng ký sẽ dừng với lỗi `'/etc/rhsm/rhsm.conf' not found, cannot configure subscription-manager`.
- Script đăng ký không ghi phần proxy: nếu Satellite mới không đi qua proxy cũ, tự xoá `proxy_hostname`/`proxy_port` cũ trong `rhsm.conf` (ví dụ `subscription-manager config --server.proxy_hostname="" --server.proxy_port=""` — lệnh này chưa chạy trong phiên kiểm chứng).
- Host từng cài gói `katello-ca-consumer-*`: gỡ gói này mới trả `rhsm.conf` về mặc định Red Hat.

### 2. Trên Satellite (tuỳ chọn): xoá bản ghi host cũ

`unregister` không xoá host trên Satellite khi setting **Delete Host upon unregister** đang tắt. Muốn bản ghi sạch:

```bash
hammer host delete --name "<HOST_FQDN>"
```

### 3. Đăng ký lại

**Hosts → Register Host** → chọn Organization, Location, Activation Key → chạy lệnh sinh ra trên host bằng root.

- Host còn đăng ký sẽ báo `This system is already registered. Use --force to override`. Bật **Force** trong form: lệnh sinh ra tự unregister, clean, đăng ký lại và dùng lại bản ghi host cũ. Bằng hammer: `hammer host-registration generate-command ... --force true` (kiểm `--help` trước, chưa chạy thử cú pháp này).

### 4. Kiểm tra

```bash
subscription-manager identity
dnf repolist
```

## Nguồn

- Red Hat Satellite 6.19, Managing hosts, chương Registering hosts to Satellite (mục 6.5: katello-ca-consumer deprecated) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19/html/managing_hosts/index
- Chạy thử trên lab Satellite 6.15 / RHEL 9.4 (phiên Claude Code 2026-11-25, dc97f4ce): đăng ký lại thành công, host nhận ID mới; hành vi của script đăng ký có được bằng cách đọc script trên lab.
