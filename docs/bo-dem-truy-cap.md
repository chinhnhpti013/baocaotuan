# Bộ đếm lượt truy cập & người đang xem

Bộ đếm hiện ở **góc dưới bên trái mọi màn hình** (đăng nhập · chọn nguồn dữ liệu ·
dashboard), chữ mờ gần bằng nền, di chuột vào thì rõ hẳn:

```
● Lượt truy cập: 1.248 · Đang xem: 3
```

Trang là HTML tĩnh chạy trên GitHub Pages nên không có máy chủ riêng để nhớ con số
chung — phải gửi ra một nơi lưu bên ngoài. Nơi đó là **Firebase Realtime Database**
(Google), chọn vì "đang xem" cần biết lúc nào một máy **rời** trang, việc mà chỉ
một kết nối thường trực (`onDisconnect`) làm chính xác được.

> Bộ đếm chỉ gửi đi đúng 2 thứ: **một nhịp "tôi đang mở trang"** và **lệnh cộng 1
> vào tổng lượt**. Không gửi dữ liệu hồ sơ, tên người dùng hay bất cứ nội dung nào
> của báo cáo.

Chưa thiết lập xong thì bộ đếm **không hiện số sai**: nó tự chuyển về chế độ cục bộ,
chỉ đếm số lần mở trang trên chính máy đó và ghi rõ hậu tố *(máy này)*.

---

## Thiết lập (làm một lần, khoảng 5 phút)

> Giao diện Firebase Console thay đổi theo thời gian. Các tên nút dưới đây lấy từ
> tài liệu chính thức của Google (bản đọc ngày 23/11/2026). Nếu màn hình của
> anh/chị hiện chữ khác, cứ tìm nút có **nghĩa** tương đương — thứ tự các bước thì
> vẫn như vậy.

### Bước 1 — Tạo project

1. Mở <https://console.firebase.google.com> và đăng nhập bằng tài khoản Google
   (dùng luôn tài khoản đang giữ file Google Sheet dữ liệu cũng được).
2. Bấm nút tạo project mới — tuỳ phiên bản giao diện, nút ghi **Create a project**,
   **Add project** hoặc **Get started with a Firebase project**.
3. Đặt tên project, ví dụ `pti-qn-dashboard`. Firebase tự sinh một **Project ID**
   kèm theo (dạng `pti-qn-dashboard-a1b2c`); **chỉ sửa được ở bước này**, sau khi
   tạo xong là cố định. Không cần sửa, để nguyên là được.
4. Nếu Console hỏi thêm về **Google Analytics** hoặc **Gemini in Firebase**: bộ đếm
   không dùng tới, tắt đi cho gọn.
5. Project mới mặc định ở gói **Spark** (miễn phí). Không cần nâng lên Blaze, cũng
   không cần nhập thẻ.

### Bước 2 — Bật Realtime Database

1. Menu bên trái, mở nhóm **Databases & Storage** (giao diện cũ hơn gọi là
   **Build**) → chọn **Realtime Database**.

   ⚠️ **Đừng chọn nhầm Cloud Firestore.** Firestore thường được Console mời chào
   trước và cũng là "database", nhưng bộ đếm viết cho **Realtime Database** — chọn
   nhầm thì trang không chạy được.

2. Bấm **Create database**.
3. **Chọn vùng đặt máy chủ.** Realtime Database chỉ có **3 vùng**:

   | Vùng | Mã | Đuôi đường dẫn database |
   |---|---|---|
   | **Singapore** ← nên chọn | `asia-southeast1` | `….asia-southeast1.firebasedatabase.app` |
   | Iowa (Mỹ) | `us-central1` | `….firebaseio.com` |
   | Bỉ | `europe-west1` | `….europe-west1.firebasedatabase.app` |

   Chọn **Singapore** cho gần Việt Nam. ⚠️ **Vùng không đổi được sau khi tạo** —
   muốn đổi thì phải tạo database khác.

4. **Chọn chế độ bảo mật**: chọn **Locked mode** (khoá hết). Bước 4 sẽ thay bằng bộ
   quy tắc riêng của bộ đếm.

   *Nếu Console chỉ cho chọn **Test mode** thì cũng không sao — nhưng Test mode mở
   toang cho mọi người ghi và tự hết hạn sau khoảng 30 ngày, nên phải làm Bước 4
   ngay, đừng để vậy.*

5. Bấm **Done** (hoặc **Enable**).

### Bước 3 — Lấy đường dẫn database và dán vào trang

Sau khi tạo xong, ngay đầu tab **Data** của Realtime Database có một dòng đường dẫn
dạng:

```
https://pti-qn-dashboard-default-rtdb.asia-southeast1.firebasedatabase.app
```

Copy nguyên dòng đó (không lấy phần sau dấu `/` nếu có) rồi dán vào `index.html`,
hằng `VC_DB_URL` — tìm chữ `VC_DB_URL` trong file:

```js
const VC_DB_URL = 'https://pti-qn-dashboard-default-rtdb.asia-southeast1.firebasedatabase.app';
```

Hoặc gửi đường dẫn đó cho Claude dán hộ. Dán xong thì commit và push lên `main` để
GitHub Pages cập nhật.

### Bước 4 — Đặt quy tắc bảo mật

1. Vẫn ở màn Realtime Database, mở tab **Rules** (có nơi ghi **Security Rules**).
2. Xoá hết nội dung đang có, dán bộ quy tắc sau:

```json
{
  "rules": {
    ".read": false,
    ".write": false,
    "stats": {
      "views": {
        ".read": true,
        ".write": "newData.isNumber() && ((!data.exists() && newData.val() == 1) || (data.exists() && newData.val() == data.val() + 1))"
      }
    },
    "online": {
      ".read": true,
      "$phien": {
        ".write": true,
        ".validate": "newData.hasChild('t') && newData.child('t').isNumber() && newData.child('t').val() <= now + 60000"
      }
    }
  }
}
```

3. Bấm **Publish**. Quy tắc mới cần **vài phút** để có hiệu lực hoàn toàn.

Ý nghĩa bộ quy tắc:

- Ai cũng **đọc** được 2 con số (trang công khai thì buộc phải vậy), nhưng chỉ đọc
  được đúng 2 nhánh này — phần còn lại của database khoá kín.
- Tổng lượt: lần ghi đầu tiên **bắt buộc bằng 1**, từ đó về sau mỗi lần ghi **chỉ
  được cộng đúng 1**. Không ai đặt bừa một số khác, không ai xoá hay hạ về 0 từ
  bên ngoài được.
- Mỗi phiên xem chỉ ghi được **một mốc thời gian**, và mốc đó không được ở tương
  lai quá 1 phút — chặn việc cắm một phiên "sống mãi" để thổi phồng số người đang xem.

> ⚠️ **Bản quy tắc đầu tiên (23/11/2026) có kẽ hở, đừng dùng lại**: nhánh
> `stats/views` khi còn **rỗng** thì cho ghi bất kỳ số nào ≥ 0, nên lần ghi đầu có
> thể bị đặt thành một con số bịa. Bản trên đã siết: rỗng thì chỉ nhận đúng số 1.
>
> **Hệ quả cần biết**: với bộ quy tắc này, tổng lượt **không thể sửa hay xoá từ bên
> ngoài** — kể cả bởi chính mình. Muốn đặt lại về 0 thì **bắt buộc** vào Firebase
> Console ▸ tab **Data** ▸ xoá nhánh `stats`. Đây là chủ ý: con số đếm không nên
> sửa được qua mạng.

### Bước 5 — Kiểm tra

Mở trang trên GitHub Pages: góc dưới trái phải hiện số và **không còn chữ
*(máy này)***. Mở thêm một tab nữa thì "Đang xem" lên 2; đóng tab đi thì sau vài
giây xuống lại 1. Trong Console, tab **Data** sẽ thấy 2 nhánh `stats` và `online`
tự sinh ra.

Nếu vẫn còn chữ *(máy này)*: mở Console của trình duyệt (F12) xem dòng cảnh báo
`[bộ đếm]` — thường là dán thiếu/thừa ký tự trong đường dẫn, hoặc quên bấm
**Publish** ở Bước 4.

---

## Những điều nên biết

| | |
|---|---|
| **Chi phí** | Miễn phí. Gói Spark cho 100 kết nối cùng lúc và 10 GB truyền/tháng — bộ đếm này dùng vài KB mỗi phiên. |
| **Đếm thế nào** | Mỗi lần **mở trang** +1, kể cả cùng một người tải lại trang (F5). Đây là cách người dùng đã chọn. |
| **"Đang xem" chính xác đến đâu** | Đóng tab là mất khỏi danh sách gần như tức thì. Máy sập hoặc mất mạng đột ngột thì sau **2,5 phút** không còn được tính; bản ghi chết được dọn sau 15 phút. |
| **Mở bằng file:// ở máy** | Vẫn chạy (Firebase dùng websocket, không vướng CORS như CSV của Google Sheet). |
| **Mất mạng** | Hai con số chuyển về `—` chứ không giữ số cũ. |
| **Muốn tắt bộ đếm** | Xoá nội dung hằng `VC_DB_URL` (để lại `''`) — bộ đếm về chế độ cục bộ. Muốn ẩn hẳn thì đặt `#visit-counter{display:none}`. |
| **Đặt lại tổng lượt về 0** | Firebase Console ▸ tab **Data** ▸ xoá nhánh `stats`. Lần mở trang sau sẽ đếm lại từ 1. ⚠ Quy tắc cố tình **không cho** sửa/xoá con số này từ bên ngoài, nên đây là cách duy nhất. |

## Chỗ sửa trong code

| Thành phần | Vị trí trong `index.html` |
|---|---|
| Hằng `VC_DB_URL`, `VC_BEAT`, `VC_TTL` | đầu khối "BỘ ĐẾM LƯỢT TRUY CẬP" trong `<script>` |
| Hàm `vcInit` · `vcStart` · `vcPaint` · `vcTheme` · `vcLocalMode` | cùng khối đó |
| Khối hiển thị `#visit-counter` | ngay trước `</body>` (ngoài `#dashboard` để hiện ở mọi màn hình) |
| Màu sắc, vị trí, cỡ chữ | CSS `#visit-counter` (đặt ngay sau `footer{}`) |
| Đổi tông sáng/tối theo màn hình | `vcTheme(id)` gọi trong `show(id)` |
