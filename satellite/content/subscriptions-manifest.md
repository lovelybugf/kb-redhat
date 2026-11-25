---
title: Subscription manifest, Simple Content Access và subscription allocation
tags: [howto, satellite, registration, hammer]
product: Red Hat Satellite 6.19
source: "Red Hat Satellite 6.19 — Managing content, chương 2 (PDF tr. 13–19); Overview, concepts, and deployment considerations, chương 4, mục 11.5, 12.4 (PDF tr. 16–17, 45–46, 54–56); Administering, phụ lục F.7 (PDF tr. 140–143); kinh nghiệm triển khai thực tế"
reliability: official
updated: 2026-11-25
owner: lovelybugf
---
**TL;DR:** Mỗi Organization trong Satellite cần đúng một subscription manifest (một subscription allocation) để lấy chứng chỉ truy cập Red Hat CDN và biết được phép bật repository nào. Simple Content Access đặt ở cấp Organization nên host không phải attach subscription; khi gia hạn hay đổi số lượng thì Refresh hoặc import lại manifest, tuyệt đối không Delete.

## Khái niệm

| Thành phần | Ý nghĩa |
|---|---|
| Subscription allocation | Một "phần" subscription của tài khoản Red Hat được tách ra cho một Satellite/Organization, quản lý trên Red Hat Hybrid Cloud Console |
| Manifest | File `.zip` xuất từ allocation, chứa danh sách subscription và chứng chỉ X.509 để Satellite xác thực với `cdn.redhat.com` |
| Organization | Mỗi Organization cần một allocation/manifest riêng; nhiều Organization = nhiều manifest, có thể thuộc các tài khoản Red Hat khác nhau |
| Simple Content Access (SCA) | Chế độ truy cập nội dung theo Organization: host đăng ký xong là dùng được mọi repository có trong content view environment của nó, không gắn subscription cụ thể |
| Future-dated subscription | Subscription có ngày bắt đầu trong tương lai; thêm vào manifest trước khi bản cũ hết hạn để không bị gián đoạn truy cập repository |

Chuỗi quan hệ nên nhớ khi vận hành: Manifest → Product → Repository → (sync) Library → Content View version → Lifecycle Environment → Activation Key → Host. Manifest chỉ quyết định "được phép dùng gì"; host thấy gì do content view environment quyết định (xem [Content View và Lifecycle Environment](content-views-and-lifecycle.md)).

## Simple Content Access

- SCA được đặt trên **Organization**, không phải trên manifest: import manifest mới không làm đổi trạng thái SCA của Organization.
- Với SCA, cột Consumed ở **Content → Subscriptions** không phản ánh số host đang dùng; muốn đối soát số lượng thì dùng báo cáo (mục "Theo dõi mức sử dụng" bên dưới).
- Kiểm tra trên host:

```bash
subscription-manager status      # dòng "Content Access Mode is set to Simple Content Access"
subscription-manager identity    # environment (LE/CV) host đang dùng
```

- Subscription tính theo hypervisor (ví dụ RHEL for Virtual Datacenters) cần [virt-who](virt-who.md) để báo cáo quan hệ hypervisor – máy ảo; cột **Requires Virt-Who** trong Content → Subscriptions cho biết subscription nào cần.

## Tạo manifest

| Satellite | Nơi tạo manifest | Manifest khi tải về |
|---|---|---|
| Connected | Red Hat Hybrid Cloud Console (Subscriptions → Manifests) | Rỗng, chưa có subscription: import vào Satellite rồi thêm subscription ngay trong Satellite (mục bên dưới) |
| Disconnected | Red Hat Customer Portal (Subscription Central) | Đã chứa subscription; mọi thay đổi phải làm trên Portal rồi tải file mới |

Khi tạo manifest nên đưa vào: subscription cho Satellite Server (nếu Satellite disconnected hoặc tự đăng ký vào chính nó), cho mọi Capsule, và cho mọi sản phẩm Red Hat sẽ quản lý; ghi lại ngày hết hạn để lên kế hoạch gia hạn.

## Import manifest

Web UI: chọn đúng Organization → **Content → Subscriptions → Manage Manifest → Choose File** → chọn file `.zip`.

Hammer (chạy trên Satellite Server):

```bash
hammer subscription upload \
  --file ~/manifest_file.zip \
  --organization "<ORG>"
hammer subscription list --organization "<ORG>"
```

Manifest lấy từ Hybrid Cloud Console (connected) còn rỗng: tiếp tục thêm subscription.

## Thêm, bớt subscription trong allocation

- Thêm: **Content → Subscriptions → Add Subscriptions** → nhập số lượng ở cột **Quantity to Allocate** → **Submit**.
- Bớt: tick subscription cần gỡ → **Delete** → xác nhận. Thao tác này chỉ gỡ subscription khỏi allocation, khác với xóa manifest.
- Tìm nhanh subscription khi danh sách dài: ô Search trên trang Subscriptions hỗ trợ điều kiện lọc.

## Refresh và cập nhật manifest

Phải refresh mỗi khi allocation thay đổi: gia hạn, đổi số lượng, mua thêm subscription.

- Web UI: **Content → Subscriptions → Manage Manifest → Refresh**.
- Hammer (lệnh lấy từ `hammer subscription --help`, không có trong tài liệu 6.19):

```bash
hammer subscription refresh-manifest --organization "<ORG>"
hammer subscription manifest-history --organization "<ORG>"
```

- Refresh trực tiếp cần Satellite liên lạc được với Red Hat Portal: setting **Subscription connection enabled** (Administer → Settings → Content) phải là Yes. Satellite disconnected: tải manifest mới từ Portal rồi import đè.
- Setting liên quan: **Manifest refresh timeout** (mặc định 1200 giây), **Expire soon days** (mặc định 120 ngày, số ngày trước khi hết hạn thì Web UI nhắc gia hạn).

## Cấu hình nguồn CDN của Organization

Tab **CDN Configuration** trong Manage Manifest quyết định Organization lấy nội dung Red Hat từ đâu:

| Kiểu | Dùng khi | Hammer |
|---|---|---|
| Red Hat CDN | Mặc định, Satellite ra được `cdn.redhat.com` (trực tiếp hoặc qua proxy) | `--type=redhat_cdn` |
| Custom CDN | Có web server nội bộ mirror đúng cấu trúc thư mục của Red Hat CDN | `--type=custom_cdn --url <URL>` |
| Network Sync | Satellite downstream kéo nội dung từ Satellite upstream qua mạng | `--type=network_sync` |
| Export Sync | Satellite air-gapped, chỉ nhận nội dung bằng import | `--type=export_sync` |

```bash
hammer organization configure-cdn --name="<ORG>" --type=redhat_cdn
```

Chi tiết Network Sync và Export Sync: [Đồng bộ giữa các Satellite](inter-satellite-sync.md).

## Theo dõi mức sử dụng subscription

| Kịch bản | Cách làm |
|---|---|
| Connected | **Red Hat Lightspeed → Inventory Upload**, bật Automatic Inventory Upload; dịch vụ Subscriptions trên Hybrid Cloud Console tính usage. Có thể che hostname/IP (obfuscation) nếu dữ liệu nhạy cảm |
| Disconnected | Sinh báo cáo cục bộ `foreman-rake rh_cloud_inventory:report:generate` (JSON) hoặc report template **Host - Installed Products** (Monitor → Reports → Report Templates); capacity lấy trên Hybrid Cloud Console, tự tính utilization |

Đặt system purpose (SLA, usage, role) cho host qua Activation Key để báo cáo chính xác hơn.

## Lưu ý và lỗi hay gặp

- **Không Delete manifest** trên Satellite hay trên Portal: toàn bộ host và Activation Key mất quyền truy cập nội dung. Đổi subscription thì Refresh hoặc import file mới.
- Sau khi đổi allocation mà sync báo lỗi xác thực với CDN: Refresh manifest rồi sync lại.
- Proxy ra Internet không được giải mã SSL (SSL inspection) với `subscription.rhsm.redhat.com` và `cdn.redhat.com`, vì Satellite xác thực bằng chứng chỉ client trong manifest.
- Manifest có chứng chỉ riêng có hạn dùng; Web UI hiển thị thông báo trước khi manifest hết hạn — đưa việc kiểm tra vào checklist định kỳ.
- Trong triển khai thực tế, cách tách môi trường DEV/PROD bằng Lifecycle Environment và Activation Key trong cùng một Organization đủ dùng; chỉ tạo thêm Organization khi cần tách tài khoản Red Hat hoặc tách quyền quản trị.
- Trong triển khai thực tế (đã kiểm chứng): sau khi import manifest, `subscription-manager status` trên host báo đúng chế độ SCA, và host chỉ cần Activation Key trỏ tới đúng content view environment là dùng được repository.

## Liên quan

- [Repository Red Hat, custom repository và đồng bộ](repositories-and-sync.md)
- [Content View và Lifecycle Environment](content-views-and-lifecycle.md)
- [virt-who cho subscription máy ảo](virt-who.md)
- [Đồng bộ nội dung giữa các Satellite](inter-satellite-sync.md)
- [Activation Key chỉ áp dụng lúc đăng ký](../hosts/activation-key-registration-only.md)
- [Cài Satellite disconnected](../installation/install-satellite-disconnected.md)

## Nguồn

- Red Hat Satellite 6.19, Managing content, chương 2 Managing Red Hat subscriptions (PDF tr. 13–19) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Overview, concepts, and deployment considerations, chương 4 Expense management (PDF tr. 16–17), mục 11.5 (PDF tr. 45–46), mục 12.4 Satellite Server with multiple manifests (PDF tr. 54–56) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Installing Satellite Server in a disconnected network environment, mục 3.9, 5.2–5.4, phụ lục B–C (PDF tr. 22–24, 30–35, 48–49) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Red Hat Satellite 6.19, Administering Red Hat Satellite, phụ lục F.7 Content management settings (PDF tr. 140–143) — https://docs.redhat.com/en/documentation/red_hat_satellite/6.19
- Kinh nghiệm triển khai thực tế (ẩn danh), 2026: import manifest, kiểm tra SCA trên host, tách DEV/PROD trong một Organization; lệnh `refresh-manifest`/`manifest-history` lấy từ hammer help, chưa đối chiếu tài liệu.
