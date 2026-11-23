---
title: Tắt Red Hat Lightspeed (Insights) trên Satellite và host
tags: [howto, satellite, lightspeed, registration, hammer]
product: Red Hat Satellite 6.18
source: "Red Hat Satellite 6.18 — Managing hosts, mục 6.2.1, 14.3, 14.6; Administering, Chương 14"
reliability: official
updated: 2026-11-23
owner: lovelybugf
---
**TL;DR:** Đặt global parameter boolean `host_registration_insights` và `host_registration_insights_inventory` bằng `false` để host đăng ký mới không cài và không gửi dữ liệu Lightspeed. Sau đó tắt Sync Automatically ở Red Hat Lightspeed → Recommendations; host đã đăng ký chạy `insights-client --unregister` và tắt `insights-client.timer`; nếu dùng Lightspeed in-Satellite thì chạy `satellite-installer --iop-ensure absent`.

## 1. Chặn cho host đăng ký mới — global parameters

**Configure → Global Parameters** → tạo (hoặc sửa nếu đã có):

| Name | Type | Value | Tác dụng |
|---|---|---|---|
| `host_registration_insights` | boolean | `false` | Không cài Insights client khi đăng ký; Satellite không chuyển dữ liệu Insights client của host lên dịch vụ Lightspeed |
| `host_registration_insights_inventory` | boolean | `false` | Loại host khỏi báo cáo inventory tải lên Red Hat Hybrid Cloud Console |

Kiểu tham số phải là **boolean**. Tham số đặt được ở mọi cấp (global, organization, location, host group, host); đặt ở global để áp dụng toàn hệ thống.

Cách làm bằng hammer (kiểm tra `hammer global-parameter set --help` có tuỳ chọn `--parameter-type` trên phiên bản đang dùng):

```bash
hammer global-parameter set --name host_registration_insights --value false --parameter-type boolean
hammer global-parameter set --name host_registration_insights_inventory --value false --parameter-type boolean
```

## 2. Tắt đồng bộ recommendations

**Red Hat Lightspeed → Recommendations** → tắt **Sync Automatically**.

## 3. Host đã đăng ký trước đó

Global parameter chỉ tác động lúc đăng ký, nên host cũ phải xử lý trực tiếp. Chạy bằng root trên từng host (hoặc hàng loạt qua job Remote Execution):

```bash
insights-client --unregister
systemctl disable --now insights-client.timer
systemctl is-enabled insights-client.timer     # mong đợi: disabled
```

## 4. Nếu đã bật Red Hat Lightspeed in Satellite

Chạy bằng root trên Satellite Server:

```bash
satellite-installer --iop-ensure absent
```

## Liên quan

- [Activation Key chỉ áp dụng lúc đăng ký](activation-key-registration-only.md) — cùng nguyên tắc: cấu hình lúc đăng ký không tự áp lên host cũ.
- [systemd — timer](https://github.com/lovelybugf/kb-base-devops/blob/main/linux/systemd-basics.md)

## Nguồn

- Red Hat Satellite 6.18, Managing hosts, mục 6.2.1 (tham số đăng ký), 14.3, 14.6 — https://docs.redhat.com/en/documentation/red_hat_satellite/6.18/html/managing_hosts/index
- Red Hat Satellite 6.18, Administering Red Hat Satellite, Chương 14 — https://docs.redhat.com/en/documentation/red_hat_satellite/6.18/html/administering_red_hat_satellite/index
- Ý nghĩa hai tham số và lệnh `--iop-ensure absent` đối chiếu thêm với bản 6.19 (Managing hosts; Administering) ngày 2026-11-23.
