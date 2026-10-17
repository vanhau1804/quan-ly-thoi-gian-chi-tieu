# Quản Lý Thời Gian & Chi Tiêu

Ứng dụng cá nhân quản lý công việc, chi tiêu (có hạn mức theo tháng), nhắc nhở và tổng kết. Giao diện tiếng Việt, chạy trên điện thoại.

## Cấu trúc thư mục

| Thư mục | Nội dung |
|---|---|
| `index.html` | Bản app chạy được (một file, React + Tailwind qua CDN, dữ liệu lưu trong trình duyệt). Nằm ở thư mục gốc để Vercel/GitHub Pages tự phục vụ. |
| `docs/thiet-ke-ung-dung.md` | Tài liệu thiết kế v2: chọn nhạc chuông, báo thức khi đóng app/tắt màn hình, khóa PIN 4 số, chia sẻ tải về, kiểm tra bảo mật. |
| `design/` | Bản mô phỏng giao diện v2 (các màn hình `.dc.html` và `canvas.json`). Đây là bản vẽ để duyệt, không phải app. |

## Trạng thái

- `index.html` đã chạy thử các luồng chính: thêm việc, thêm chi tiêu, đặt hạn mức, thêm nhắc nhở, đặt/khóa/mở khóa PIN, xuất file sao lưu JSON.
- Nhạc chuông: mỗi nhắc nhở chọn được 1 trong 6 chuông có sẵn hoặc file nhạc riêng (tối đa 5 MB, lưu trong trình duyệt), nghe thử, lặp 1–3 lần, và đổi lại sau khi đã tạo. File nhạc riêng không nằm trong file sao lưu JSON.
- Nhắc nhở trên bản web chỉ reo khi trang đang mở. Báo thức đúng giờ khi đóng app và tắt màn hình cần bản Android native (xem tài liệu thiết kế).
- PIN ở bản web chỉ là khóa giao diện, dữ liệu chưa được mã hóa.
- Các bước triển khai bản v2 chỉ làm sau khi bản thiết kế được duyệt.

## Chạy thử

Mở `index.html` bằng trình duyệt (cần có Internet để tải thư viện từ CDN).
