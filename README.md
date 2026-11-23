# kb-redhat

Kho tri thức sản phẩm Red Hat (tiếng Việt): Satellite, OpenShift, IdM, RHEL, Ansible/AAP.

- Bắt đầu từ [index.md](index.md) → `_index.md` của sản phẩm (có phiên bản đang dùng + link tài liệu chính thức).
- Viết trang theo chuẩn [kb-core](https://github.com/lovelybugf/kb-core/blob/main/index.md); tạo trang mới:

```bash
python kb-core/tools/new_page.py kb-redhat/satellite/<ten-trang> "Tiêu đề" --tags howto,satellite --product "Red Hat Satellite 6.19"
python kb-core/tools/kb_index.py kb-redhat
python kb-core/tools/kb_lint.py kb-redhat
```

Kho chứa kiến thức sản phẩm, không chứa thông tin khách hàng (FQDN, IP, tài khoản, manifest). Tài liệu dự án để ở nơi lưu trữ riêng của dự án.
