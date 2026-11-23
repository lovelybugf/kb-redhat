---
title: Download policy của repository (on_demand và immediate)
tags: [reference, satellite, download-policy, capsule, pulp, hammer]
product: Red Hat Satellite 6.16
source: "Red Hat Satellite 6.16 — Administering, Phụ lục B.7; Managing content, mục 4.10–4.11"
reliability: official
updated: 2026-11-23
owner: lovelybugf
---
**TL;DR:** Mặc định repo Red Hat dùng `on_demand` (chỉ tải metadata, gói được tải khi host/Capsule yêu cầu), repo custom dùng `immediate`. Proxy ra Internet chỉ mở theo khung giờ thì chuyển repo sang `immediate` và sync trong khung giờ đó; đổi giá trị mặc định không ảnh hưởng repo đã có, Capsule để Inherit/On Demand, và dung lượng `/var/lib/pulp` sẽ tăng.

## Các policy

| Policy | Trên Satellite | Trên Capsule |
|---|---|---|
| Immediate | Tải toàn bộ metadata và gói khi sync | Tải toàn bộ khi sync Capsule |
| On Demand | Chỉ tải metadata; gói được tải và lưu khi Capsule/host yêu cầu | Chỉ metadata; gói lấy từ Satellite khi host yêu cầu |
| Inherit | — | Theo policy của repo tương ứng trên Satellite |
| Streamed | — | Capsule chỉ làm proxy, không lưu gói |

On Demand (lazy sync) chỉ dùng cho repo Yum.

## Giá trị mặc định

Administer → Settings → thẻ **Content**:

| Setting | Tên trong hammer | Mặc định |
|---|---|---|
| Default Red Hat Repository download policy | `default_redhat_download_policy` | `on_demand` |
| Default Custom Repository download policy | `default_download_policy` | `immediate` |

Đổi giá trị mặc định **chỉ áp dụng cho repo tạo/bật sau đó**; repo đã có giữ nguyên policy, phải đổi từng repo.

## Tình huống: proxy chỉ mở theo khung giờ

Với `on_demand`, gói chỉ được tải từ CDN khi có host/Capsule yêu cầu — thường rơi ngoài khung giờ proxy mở, khiến cài/cập nhật gói trên host bị lỗi. Cách xử lý:

1. Chuyển các repo đang dùng sang `immediate`:

```bash
hammer repository list --organization "<ORG>"
hammer repository update --id <REPO_ID> --download-policy immediate
# hoặc theo tên:
hammer repository update \
  --download-policy immediate \
  --name "<REPO_NAME>" \
  --organization-label <ORG_LABEL> \
  --product "<PRODUCT>"
```

2. Đặt Sync Plan chạy trong khung giờ proxy mở để Satellite tải đủ gói về.
3. Capsule: để download policy **Inherit** hoặc **On Demand** — Capsule lấy gói từ Satellite (đã có đủ sau khi sync `immediate`), không cần ra Internet.
4. Repo bật thêm sau này cũng cần `immediate` → đổi luôn mặc định:

```bash
hammer settings set --name default_redhat_download_policy --value immediate
```

5. Kiểm tra dung lượng trước và sau khi sync — `immediate` tải toàn bộ gói nên `/var/lib/pulp` tăng mạnh:

```bash
df -h /var/lib/pulp
```

Cần mở rộng: xem [LVM — tạo và mở rộng volume](../../kb-base-devops/linux/storage-lvm.md).

## Kiểm tra

```bash
hammer repository info --id <REPO_ID>      # xem trường Download Policy
```

## Liên quan

- [Activation Key chỉ áp dụng lúc đăng ký](activation-key-registration-only.md)

## Nguồn

- Red Hat Satellite 6.16, Administering Red Hat Satellite, Phụ lục B.7 (Content settings) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.16/html/administering_red_hat_satellite/index
- Red Hat Satellite 6.16, Managing content, mục 4.10–4.11 — https://docs.redhat.com/en/documentation/red_hat_satellite/6.16/html/managing_content/index
- Bảng policy, tên setting và cú pháp hammer đối chiếu thêm với bản 6.19 (Managing content, Chương 6, mục 6.1–6.5; Administering, phụ lục Content settings) ngày 2026-11-23.
