# Thiết kế: Quản Lý Thời Gian & Chi Tiêu (phiên bản 2)

Trạng thái: **Bản thiết kế chờ duyệt.** Chưa triển khai. Sau khi duyệt mới làm các bước tiếp theo.

---

## 0. Tóm tắt quyết định

| Hạng mục | Đề xuất | Lý do ngắn gọn |
|---|---|---|
| Nền tảng | **Android trước** (APK), iOS giai đoạn sau | Chỉ Android cho phép báo thức đúng giờ khi màn hình tắt, app đóng. iOS bị giới hạn bởi hệ điều hành. |
| Công nghệ | Giữ React, bọc bằng **Capacitor** | Tái sử dụng phần lớn code hiện có, truy cập được AlarmManager và Keystore. |
| Báo thức | Native Android: `AlarmManager.setAlarmClock` + foreground service | Đây là API dành cho báo thức, đáng tin cậy hơn `setInterval` của web. |
| Nhạc chuông | 7 chuông đóng gói sẵn + cho phép chọn file âm thanh từ máy | Đáp ứng yêu cầu 1 |
| PIN | 4 số, hash PBKDF2 + salt, lưu trong Android Keystore, giới hạn số lần sai | Đáp ứng yêu cầu 3, có ghi rõ giới hạn |
| Chia sẻ | Trang tải APK + bản web trên **Cloudflare Pages** (HTTPS, miễn phí) | Ổn định hơn chạy trên máy cá nhân qua cloudflared |
| Dữ liệu | Lưu trên từng máy, không có máy chủ chung | Không có backend thì không thể "dùng chung dữ liệu" (xem mục 7) |

---

## 1. Ràng buộc kỹ thuật thực tế (cần đọc trước)

Yêu cầu 2 ("báo đúng giờ khi không bật app và màn hình tắt") **không thể làm được trong trang web thuần**:

- Trình duyệt tạm dừng JavaScript khi tab bị ẩn hoặc màn hình khóa. `setInterval` sẽ bị dừng.
- Web Push (thông báo từ máy chủ) cũng không đảm bảo reo chuông to. Thông báo bị tắt khi chế độ im lặng hoặc tập trung đang bật, và không có âm báo thức riêng.
- Trên iOS Safari, Web Push chỉ hoạt động khi app được cài vào màn hình chính (iOS 16.4 trở lên).

Vì vậy, để đạt yêu cầu 2 cần làm **ứng dụng gốc (native)**. Ba phương án:

| Phương án | Báo khi tắt màn hình | Chi phí | Khuyến nghị |
|---|---|---|---|
| A. Capacitor + plugin báo thức native (Android) | Có, đáng tin cậy | Trung bình, cần build APK | **Chọn** |
| B. PWA + Web Push | Không đảm bảo, không có chuông báo thức | Thấp | Không đủ yêu cầu |
| C. Giữ web hiện tại | Chỉ khi tab đang mở | Thấp | Không đạt yêu cầu 2 |

**Giới hạn đã biết của phương án A:**
- Một số hãng (Xiaomi, Oppo, Vivo, Huawei) có chế độ tiết kiệm pin tự động tắt báo thức của app. Người dùng phải tự tắt tối ưu hóa pin cho app. Ứng dụng sẽ hướng dẫn bước này khi cài đặt lần đầu.
- Nếu người dùng buộc dừng app (Force Stop) trong Cài đặt, Android sẽ hủy toàn bộ báo thức đã đặt. App phải đặt lại báo thức khi mở lần sau. Không có cách nào khác.
- Android 14 trở lên yêu cầu quyền báo thức chính xác. App báo thức dùng `setAlarmClock` nên không cần xin quyền riêng ở hầu hết trường hợp, nhưng cần kiểm tra lại trên từng phiên bản khi test.
- iOS: app không thể reo chuông to khi đang bật chế độ im lặng nếu không có entitlement "Critical Alerts" do Apple cấp theo yêu cầu. Đây là quyết định ngoài tầm của dự án. Giai đoạn iOS sẽ chỉ hỗ trợ thông báo có âm thanh tùy chỉnh (tối đa 30 giây).

---

## 2. Yêu cầu 1: Chọn nhạc chuông

**Chức năng**
- Thư viện 7 chuông đóng gói sẵn trong app (file âm thanh nhỏ, tối đa 5 giây mỗi đoạn).
- Chọn số lần lặp: 1 lần / 2 lần / 3 lần.
- Nghe thử trước khi chọn. Có nút dừng.
- Chọn file âm thanh từ máy (mp3, m4a, ogg), tối đa 5 MB, sao chép vào bộ nhớ riêng của app. Không dùng đường dẫn gốc vì file gốc có thể bị xóa.
- Chọn chuông riêng cho từng nhắc nhở, hoặc dùng chuông mặc định chung.
- Chuông phát qua **luồng âm báo thức** (alarm stream) của Android, nên vẫn reo khi máy ở chế độ rung hoặc "không làm phiền" (nếu người dùng cho phép app vượt qua chế độ này).

**Dữ liệu**
```
settings.defaultRingtone: "crystal" | "forest" | ... | "custom:<fileId>"
settings.ringtoneRepeat: 1 | 2 | 3
reminder.ringtone: null (dùng mặc định) | id chuông
```

**Tiêu chí chấp nhận**
- Nghe thử phát ngay, không bị trễ.
- Chuông tự chọn vẫn phát đúng khi app đã bị đóng hoàn toàn.
- File tùy chỉnh bị lỗi hoặc quá lớn thì báo lỗi rõ ràng, không làm hỏng lịch nhắc.

---

## 3. Yêu cầu 2: Báo thức đúng giờ khi app đóng, màn hình tắt

**Kiến trúc (Android)**

1. Mỗi lịch báo thức được đăng ký vào **AlarmManager** bằng `setAlarmClock(triggerTime, pendingIntent)`. Android coi đây là báo thức thật, được ưu tiên ngay cả khi máy ở chế độ tiết kiệm pin (Doze).
2. Khi đến giờ, hệ thống gọi một **BroadcastReceiver** và khởi động **foreground service** để phát chuông. Dịch vụ này hiển thị thông báo thường trực "Đang báo thức", cho phép tắt ngay từ thanh thông báo.
3. Khi máy đang khóa, dùng **full-screen intent** để hiện màn hình báo thức trên màn hình khóa (có nút "Tắt" và "Báo lại 5 phút").
4. Sau khi khởi động lại máy (`BOOT_COMPLETED`), toàn bộ báo thức được đăng ký lại từ bộ nhớ.
5. Khi múi giờ hoặc giờ hệ thống thay đổi (`TIME_SET`, `TIMEZONE_CHANGED`), đăng ký lại toàn bộ báo thức.
6. Lịch lặp hàng ngày và hàng tháng: mỗi lần reo xong, tự tính và đăng ký lần kế tiếp. Không lưu "đã reo hôm nay" trong bộ nhớ của web nữa.

**Quy tắc lập lịch (thay cho logic cũ)**
- Lịch một lần: đăng ký đúng 1 thời điểm. Sau khi reo, tự xóa hoặc đánh dấu đã xong.
- Lịch hàng ngày: giờ cố định mỗi ngày.
- Lịch hàng tháng: ngày trong tháng. Nếu tháng không có ngày đó (ví dụ ngày 31 của tháng 2), reo vào ngày cuối tháng và ghi chú rõ với người dùng.
- Không đăng ký báo thức cho thời điểm đã qua. Hiển thị cảnh báo nếu người dùng chọn giờ đã qua.

**Quyền và cài đặt cần người dùng cấp (có màn hình hướng dẫn riêng)**
- Thông báo (POST_NOTIFICATIONS, Android 13+).
- Hiển thị trên màn hình khóa (full-screen intent).
- Bỏ tối ưu hóa pin cho app (REQUEST_IGNORE_BATTERY_OPTIMIZATIONS).
- Mở hướng dẫn riêng cho các hãng có tự động dọn app (Xiaomi, Oppo...).

**Ma trận kiểm thử bắt buộc (trước khi phát hành)**

| Tình huống | Kỳ vọng |
|---|---|
| App đang mở, màn hình sáng | Reo đúng giờ |
| App ở nền | Reo đúng giờ |
| App đã vuốt khỏi danh sách gần đây | Reo đúng giờ |
| Màn hình tắt, máy khóa | Reo đúng giờ, hiện màn hình báo thức |
| Máy ở chế độ rung / không làm phiền | Reo theo cấu hình |
| Máy tắt nguồn rồi bật lại | Báo thức vẫn đăng ký lại và reo |
| Đổi múi giờ hoặc chỉnh giờ hệ thống | Đăng ký lại, reo đúng giờ mới |
| Bật chế độ tiết kiệm pin (Xiaomi/Samsung/Oppo) | Có hướng dẫn, reo sau khi người dùng đã bỏ tối ưu |
| Buộc dừng app trong Cài đặt | Báo thức bị hủy; khi mở app lại, cảnh báo và đăng ký lại |
| Nhiều báo thức cùng lúc | Reo lần lượt, không bỏ sót |

Mỗi tình huống cần test trên ít nhất 2 máy Android khác hãng và 2 phiên bản Android khác nhau.

---

## 4. Yêu cầu 3: PIN 4 số

**Chức năng**
- Tạo PIN 4 chữ số, xác nhận hai lần.
- Đổi PIN: phải nhập PIN hiện tại.
- Tắt PIN: phải nhập PIN hiện tại.
- Khóa tự động khi app ở nền quá **1 phút** (có thể chỉnh: 0, 1, 5 phút).
- Khóa ngay bằng nút khóa.
- Sau **5 lần nhập sai liên tiếp**, khóa nhập trong 30 giây. Mỗi lần sai tiếp theo, thời gian chờ tăng gấp đôi (tối đa 15 phút).
- Không hiển thị PIN khi nhập. Có độ trễ ngắn sau mỗi lần nhập sai.

**Lưu trữ**
- Không lưu PIN dạng rõ và không dùng SHA-256 đơn giản.
- Hash bằng **PBKDF2-HMAC-SHA256**, salt ngẫu nhiên 16 byte, tối thiểu 100.000 vòng lặp.
- Lưu hash và salt trong **Android Keystore** (qua plugin secure storage), không dùng localStorage.
- Bộ đếm số lần sai và thời điểm khóa cũng lưu trong bộ nhớ an toàn, để không reset được bằng cách đóng app.

**Giới hạn cần nói rõ với người dùng**
- Chỉ có 10.000 tổ hợp PIN 4 số. Nếu kẻ tấn công lấy được hash và salt (ví dụ máy đã root), có thể dò PIN trong vài phút. Vì vậy PIN chỉ là **khóa giao diện** để người khác không mở được app khi cầm máy, không phải lớp bảo vệ cho máy bị xâm nhập.
- Nếu cần bảo vệ thật sự cho dữ liệu, cần mật khẩu dài hơn hoặc sinh trắc học (vân tay / khuôn mặt). Đề xuất đưa vào giai đoạn sau, cần bạn quyết định.
- Các thông báo nhắc nhở và chuông báo thức vẫn hiện khi app bị khóa (đây là hành vi mong muốn). Nội dung trên màn hình khóa chỉ hiển thị tiêu đề, không hiện số tiền.

**Tiêu chí chấp nhận**
- Sau 5 lần sai, không thể thử tiếp trong thời gian khóa, kể cả khi đóng mở app.
- Đổi hoặc tắt PIN đều yêu cầu PIN hiện tại.
- Dữ liệu hash không xuất hiện trong file sao lưu.

---

## 5. Yêu cầu 4: Chia sẻ qua Internet để mọi người tải về dùng

**Hai kênh phát hành**

1. **Bản web (PWA)** trên Cloudflare Pages, ví dụ `tem.lopcuatoi.org`.
   - HTTPS có sẵn, miễn phí, không cần máy của bạn bật.
   - Người dùng mở bằng trình duyệt hoặc thêm vào màn hình chính.
   - Không có báo thức khi màn hình tắt (xem mục 1).
2. **File APK Android** đặt trên Cloudflare Pages (hoặc GitHub Releases) tại `tem.lopcuatoi.org/download`.
   - Người dùng tải về và cài. Cần bật "Cài đặt ứng dụng không rõ nguồn" trên điện thoại.
   - Trang tải có hướng dẫn cài bằng ảnh, và mã QR để quét từ máy tính.
   - Mỗi bản phát hành có số phiên bản và ghi chú thay đổi.

**Không khuyến nghị** chạy app trên máy cá nhân qua cloudflared (cách hiện tại của bạn) để chia sẻ lâu dài, vì máy phải luôn bật và kết nối. Cloudflared vẫn có thể dùng để test nhanh.

**Ký APK**
- Tạo keystore riêng cho app. Lưu file keystore và mật khẩu ở nơi an toàn (ngoài máy làm việc). Mất keystore thì không thể cập nhật app cho người đã cài.
- Không đưa keystore vào kho mã nguồn.

**Dữ liệu khi chia sẻ**
- Mỗi người dùng có dữ liệu riêng trên máy của họ. Không có máy chủ, nên không có "dữ liệu dùng chung".
- Nếu muốn nhiều người dùng chung một bộ dữ liệu (ví dụ cả gia đình), cần thêm backend và đăng nhập. Đây là thay đổi lớn về kiến trúc, bảo mật và chi phí. Xem mục 9.

**Cập nhật app**
- Bản web: tự cập nhật khi người dùng tải lại trang.
- APK: người dùng tải bản mới và cài đè. Cần giữ nguyên chữ ký (keystore) để cài đè được.

---

## 6. Yêu cầu 5: Bảo mật (kiểm tra chi tiết)

### 6.1. Các lỗ hổng đã biết trong bản hiện tại

| # | Vấn đề | Mức độ | Xử lý trong bản mới |
|---|---|---|---|
| S1 | Thư viện React, Babel, Tailwind tải từ CDN không có kiểm tra toàn vẹn (SRI) | Cao | Đóng gói thư viện vào app, không dùng CDN khi chạy. Nếu vẫn dùng CDN thì thêm thuộc tính `integrity`. |
| S2 | PIN hash SHA-256 không salt ngẫu nhiên, không có giới hạn số lần sai | Cao | PBKDF2 + salt ngẫu nhiên + Keystore + giới hạn lần sai (mục 4). |
| S3 | Logic khóa PIN trong bản trước có nhánh `!err \|\| true`, luôn đúng, làm hành vi khó đoán | Trung bình | Viết lại toàn bộ luồng khóa, có kiểm thử. |
| S4 | Dữ liệu (việc, chi tiêu, PIN nếu có) lưu trong localStorage dạng rõ | Trung bình | Dữ liệu nhạy cảm đưa vào lưu trữ an toàn; dữ liệu còn lại mã hóa bằng khóa lấy từ Keystore. |
| S5 | Nhập file sao lưu không kiểm tra cấu trúc, có thể ghi dữ liệu rác hoặc thực thi nội dung | Cao | Kiểm tra schema từng trường, giới hạn kích thước file (5 MB), loại bỏ trường lạ, không bao giờ dùng `eval` hay HTML từ file. |
| S6 | Hiển thị nội dung người dùng nhập (ghi chú, tiêu đề) | Thấp | React tự escape. Cấm `dangerouslySetInnerHTML`. Kiểm tra bằng test có chuỗi dạng `<script>`. |
| S7 | File sao lưu chứa toàn bộ dữ liệu, không có mã hóa | Trung bình | Tùy chọn đặt mật khẩu khi xuất; mã hóa AES-GCM với khóa từ PBKDF2. Mặc định cảnh báo người dùng không chia sẻ file này. |
| S8 | Không có chính sách quyền riêng tư | Trung bình | Viết chính sách ngắn: dữ liệu ở đâu, không gửi đi đâu, cách xóa. |
| S9 | Nội dung cảnh báo trên màn hình khóa lộ thông tin | Thấp | Chỉ hiện tiêu đề nhắc nhở, không hiện số tiền hay ghi chú chi tiêu. |
| S10 | Quyền Android khai báo thừa | Thấp | Chỉ khai báo quyền cần dùng; rà soát bằng công cụ lint trước khi phát hành. |

### 6.2. Yêu cầu bảo mật cho bản Android

- `minSdkVersion` ít nhất 26 (Android 8). Bật `allowBackup="false"` để dữ liệu nhạy cảm không bị sao lưu lên đám mây của Google.
- Bật R8/ProGuard khi build bản phát hành.
- Chặn cleartext HTTP (`usesCleartextTraffic="false"`). Mọi kết nối phải qua HTTPS.
- Không xuất component (`exported`) không cần thiết trong AndroidManifest.
- Không ghi log dữ liệu tài chính hoặc PIN.
- Khóa ký APK không đưa vào git. Thêm `.gitignore` cho keystore và file `.properties`.

### 6.3. Yêu cầu bảo mật cho bản web

- Header Content-Security-Policy: chỉ cho phép script và style từ chính trang.
- Header `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`, `Strict-Transport-Security`.
- Không có `user-scalable=no` (vẫn cho phép phóng to chữ).
- Trang tải APK phải dùng HTTPS. Cung cấp mã SHA-256 của file APK để người dùng kiểm tra file tải về không bị thay đổi.

### 6.4. Quy trình kiểm tra trước khi phát hành

1. Rà soát mã (code review) tập trung vào: nhập file, PIN, lưu trữ, quyền Android.
2. Chạy công cụ phân tích tĩnh (Android Lint, ESLint với plugin bảo mật, `npm audit`).
3. Kiểm thử thủ công các tình huống: PIN sai liên tiếp, nhập file hỏng, file rất lớn, chuỗi có ký tự HTML, đổi múi giờ.
4. Kiểm tra APK đã build: không có keystore, không có API key, không có đường dẫn nội bộ của máy phát triển.
5. Kiểm tra trang web bằng công cụ đánh giá header bảo mật.
6. Ghi lại kết quả kiểm thử và phiên bản đã phát hành.

---

## 7. Kiến trúc dữ liệu

Giữ nguyên các thực thể cũ, thêm trường để hỗ trợ tính năng mới. Có số phiên bản schema để nâng cấp về sau.

```
schemaVersion: 2

tasks[]: { id, title, description, date (YYYY-MM-DD), startTime, endTime,
           priority (low|medium|high), estimatedMinutes|null, actualMinutes|null,
           status (todo|done), createdAt, completedAt|null }

expenses[]: { id, amount (số nguyên VND, > 0), category, note (≤ 200 ký tự),
              date, time, createdAt }

limits: { "YYYY-MM": amount, ... }     // áp dụng từ tháng đó trở đi

reminders[]: { id, title (≤ 150), note (≤ 300), type (once|daily|monthly),
               date|null, dayOfMonth|null, time (HH:MM), enabled,
               ringtone|null, ringtoneRepeat|null, lastFired|null, createdAt }

settings: { defaultRingtone, ringtoneRepeat, autoLockMinutes,
            pinEnabled (chỉ cờ; hash nằm trong Keystore) }
```

**Quy tắc nhập và xuất**
- Khi nhập file: kiểm tra `schemaVersion`, kiểm tra từng trường theo kiểu và độ dài, bỏ qua trường lạ, từ chối nếu sai cấu trúc, và hiển thị số lượng bản ghi trước khi ghi đè.
- Trước khi nhập, tự động sao lưu dữ liệu hiện có để có thể hoàn tác.

---

## 8. Kế hoạch triển khai (sau khi bạn duyệt)

| Giai đoạn | Nội dung | Kết quả kiểm chứng |
|---|---|---|
| 0. Chốt | Bạn duyệt thiết kế, trả lời các câu hỏi mục 10 | Biên bản quyết định |
| 1. Nền tảng | Tách code, đóng gói thư viện, thiết lập Capacitor, keystore, CI build APK | APK cài được, mở được app |
| 2. Dữ liệu & bảo mật | Schema mới, kiểm tra nhập/xuất, lưu trữ an toàn, PIN mới | Bộ kiểm thử đơn vị cho PIN và nhập file |
| 3. Nhạc chuông | Thư viện chuông, chọn chuông, file tùy chỉnh | Nghe thử, chọn chuông, phát khi app đóng |
| 4. Báo thức native | AlarmManager, foreground service, full-screen intent, khởi động lại | Ma trận kiểm thử mục 3 |
| 5. Phát hành | Trang tải APK, bản web, header bảo mật, chính sách riêng tư | Tải về và cài được trên máy thật |
| 6. Kiểm thử & sửa lỗi | Test trên nhiều máy, nhiều hãng | Báo cáo kiểm thử |

Thời gian ước lượng cho các giai đoạn 1–5: cần xác nhận lại sau khi chốt phạm vi. Phần báo thức native và kiểm thử đa thiết bị thường chiếm nhiều thời gian nhất.

iOS không nằm trong các giai đoạn trên. Nếu cần, sẽ lập kế hoạch riêng sau khi bản Android ổn định.

---

## 9. Ngoài phạm vi phiên bản này

- Đăng nhập tài khoản và đồng bộ dữ liệu giữa nhiều thiết bị hoặc nhiều người (cần backend, cơ sở dữ liệu, bảo mật phía máy chủ).
- Thông báo đẩy từ máy chủ.
- Xác thực sinh trắc học.
- Ứng dụng iOS (giai đoạn sau).

---

## 10. Các quyết định cần bạn xác nhận

1. **Nền tảng:** đồng ý làm Android (APK) trước, bỏ qua việc báo thức trên web? Hay chấp nhận bản web không có báo thức khi màn hình tắt?
2. **Người dùng chính:** chỉ dùng cá nhân, hay chia sẻ cho nhiều người? Nếu nhiều người và cần dữ liệu chung, đây là thay đổi lớn cần cân nhắc.
3. **Thiết bị mục tiêu:** bạn và người dùng khác dùng máy hãng nào, phiên bản Android nào? Điều này quyết định số máy cần test.
4. **Chuông:** 7 chuông có sẵn đủ chưa, hay cần thêm chuông gì khác?
5. **Khóa PIN:** 4 số là đủ cho khóa giao diện, hay muốn thêm vân tay/khuôn mặt (giai đoạn sau)?
6. **Chia sẻ:** dùng domain `lopcuatoi.org` cho trang tải APK, có được không?
7. **Tên miền và tài khoản:** có tài khoản Cloudflare và Google Play Console không? (Google Play không bắt buộc; có thể phát hành qua link trực tiếp.)
8. **Dữ liệu cũ:** có cần chuyển dữ liệu từ bản web hiện tại sang bản mới không? Nếu có, file sao lưu JSON từ bản hiện tại sẽ được nhập vào phiên bản 2.

---

## Phụ lục A: Những điểm bản HTML vừa viết còn yếu

Trước khi làm bản này, cần nói rõ bản `time-expense-manager-fixed.html` vẫn còn các điểm yếu được liệt kê ở mục 6.1: thư viện tải từ CDN không có SRI (S1), PIN chỉ hash SHA-256 không salt (S2), và logic khóa PIN có nhánh rối (S3). Bản thiết kế này thay thế các điểm đó. Bản HTML chỉ nên dùng để xem thử, không nên chia sẻ rộng cho đến khi bản Android được làm xong.
