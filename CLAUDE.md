# CLAUDE.md — kb-redhat

Phạm vi: tri thức sản phẩm Red Hat — Satellite, OpenShift (OCP), IdM, RHEL, Ansible/AAP và sản phẩm khác (`other/`). Kiến thức nền trung lập (Linux, Kubernetes) nằm ở kb-base-devops.

## Thứ tự đọc
1. File này → [index.md](index.md) → `_index.md` của sản phẩm (ghi phiên bản đang dùng) → mở 2–3 trang cần thiết.
2. Không đọc toàn bộ kho; chỉ mở `raw/` khi cần đối chiếu tài liệu gốc.

## Quy ước
- Chuẩn chung ở [kb-core](https://github.com/lovelybugf/kb-core/blob/main/index.md); tag chỉ lấy từ `kb-core/standards/tags.md`.
- Trường `product` ghi đúng phiên bản đã đối chiếu (ví dụ `Red Hat Satellite 6.16`); câu hỏi về phiên bản khác → nói rõ khác biệt có thể có.
- Nguồn ghi tới chương/mục tài liệu docs.redhat.com.
- Lệnh trong code block; placeholder `<SATELLITE_FQDN>`, `<CAPSULE_FQDN>`, `<HOST_FQDN>`, `<ORG>`.
- Khi trả lời: nêu đường dẫn trang, phiên bản và mục tài liệu hãng.
- Link sang kho khác là URL `https://github.com/lovelybugf/<kho>/blob/main/<đường-dẫn>` — đọc file `../<kho>/<đường-dẫn>` trên máy, không tải web.
- Sửa trang → cập nhật `updated`, chạy `kb_index.py` + `kb_lint.py` (trong kb-core/tools), ghi 1 dòng vào [log.md](log.md).

## Kho liên quan
- Cổng vào: [kb-hub/index.md](https://github.com/lovelybugf/kb-hub/blob/main/index.md)
- Nền tảng: [kb-base-devops/index.md](https://github.com/lovelybugf/kb-base-devops/blob/main/index.md) (systemd, LVM, Kubernetes, GitOps).

## Không được
- Không ghi tên khách hàng, FQDN/IP thật, mật khẩu, manifest, token, OAuth secret của dự án.
- Không chép nguyên văn dài từ tài liệu hãng — tóm lược + lệnh + chỉ mục nguồn.
- Không viết lại kiến thức Linux/Kubernetes chung — đặt link sang kb-base-devops.
- Không sửa/xoá dòng cũ trong `log.md`.
