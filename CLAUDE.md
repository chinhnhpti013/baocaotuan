# Dự án: Báo cáo Tiến trình Bồi thường — PTISOS Quảng Ninh

## Tổ chức

**Đơn vị**: PTISOS - Văn phòng Miền Bắc - Phòng Khu vực Quảng Ninh

## Repo & trang chạy thật

| | |
|---|---|
| Repo GitHub | `https://github.com/chinhnhpti013/baocaotuan` (nhánh triển khai: `main`) |
| Trang GitHub Pages | `https://chinhnhpti013.github.io/baocaotuan/` |

> **Đổi tên repo 24/11/2026**: `xcg-dashboard` → `baocaotuan`. Link Pages cũ `…/xcg-dashboard/` nay trả **404, không redirect** (đã đo) — ai còn lưu bookmark cũ phải đổi. `git remote origin` đã trỏ sang URL mới. Tên chủ sở hữu và `GDRIVE_PUB_ID` không đổi; Firebase RTDB của bộ đếm **không** chặn theo tên miền nên không phải cấu hình lại.

## Mục đích

Tự động hóa việc tổng hợp và xuất dashboard báo cáo tiến trình bồi thường xe ô tô tại PTI Quảng Ninh từ dữ liệu file `input/Dulieu.xlsx`, hiển thị dưới dạng HTML tương tác chạy offline trong trình duyệt.

## Ngôn ngữ giao tiếp

Luôn trả lời bằng **tiếng Việt**.

## Cấu trúc dự án

```
bao-cao-tuan/
├── index.html          ← Dashboard chính (HTML thuần, không cần server)
├── assets/
│   ├── logo-ptisos.png
│   └── tro-ly-gdv-kb.js ← Bộ tri thức Trợ lý GĐV (quy trình PTI) — xem mục "Trợ lý ảo GĐV"
├── input/
│   └── Dulieu.xlsx     ← Dữ liệu nguồn, người dùng tải lên qua giao diện
├── output/
│   └── README.md
├── tools/              ← Script Node kiểm thử bộ tri thức Trợ lý GĐV
│   ├── kb-env.js       ← Nạp KB + cắt engine khỏi index.html, chạy trong vm
│   ├── check-kb.js     ← Toàn vẹn dữ liệu + không lộ email/SĐT
│   ├── test-search.js  ← tlSearch trả đúng mục
│   ├── compare-search.js ← So trước/sau khi sửa, phát hiện hồi quy
│   └── README.md
├── docs/
│   ├── Quy_trinh.md    ← Đặc tả logic báo cáo
│   ├── Quy_trinh.docx
│   └── bo-dem-truy-cap.md ← Hướng dẫn thiết lập Firebase cho bộ đếm (xem mục "Bộ đếm")
│   (docs/Kho tri thức/ ĐÃ XOÁ 23/11/2026 — 179 file / 340 MB, chuyển vào Thùng rác
│    Windows sau khi đối chiếu đủ nguồn. Bản gốc nay chỉ còn trên Google Drive —
│    xem mục "Kho tri thức trên Google Drive")
├── .claude/
│   ├── skill.md        ← Hướng dẫn Claude thêm tab / chỉnh sửa báo cáo
│   └── settings.json
└── CLAUDE.md           ← File này
```

## Nguồn dữ liệu — Dulieu.xlsx

Cột quan trọng:

| Cột | Mô tả |
|-----|-------|
| `Số HSBT` | Mã định danh hồ sơ bồi thường |
| `GĐV thụ lý` hoặc `Mã GĐV` | Mã giám định viên |
| `Trạng thái hồ sơ` | 14 giá trị (`"Đã thanh toán"`, `"Đã hủy"`, `"Chờ thanh toán"`…) — chỉ dùng để hiển thị/nhóm, **không** dùng để xác định đã giải quyết |
| `Số ngày tồn` | `= 0` → **đã giải quyết** · `> 0` → đang tồn · trống/`-`/chữ → chưa giải quyết |
| `Ngày mở HSBT` | Ngày phát sinh hồ sơ |
| `Mã nghiệp vụ` | `XO.1.x` (TNDS), `XO.4.x` (VCX) |
| `Tiền ước/duyệt BT` | Số tiền bồi thường ước tính / đã duyệt |
| `Biển số xe` | Biển kiểm soát xe |
| `Mã kênh tổng` · `Mã kênh bán` · `Mã điểm bán` | Kênh khai thác của đơn (18 mã kênh tổng: SH · 00.GARAGE · NHK · VNPOST · KHONG_KENH_KT · DKPTI…). `Mã kênh tổng` là khoá ghép với sheet doanh thu theo kênh — xem mục "Engine Đợt 3" |
| `Tên garage (thường gọi)` hoặc `Tên garage (ĐKKD)` | Tên gara/showroom sửa chữa — từ 06/2026 cột `Tên garage` cũ đã tách thành 2 cột này; báo cáo ưu tiên dùng `(thường gọi)`, fallback `(ĐKKD)` |
| `Tiền BT đã trả cho KH` | Không còn khoảng trắng cuối tên cột (khác với file cũ) — code có fallback đọc cả 2 dạng |

> ⚠️ **Chuẩn VAT lệch nhau**: `Tiền ước/duyệt BT` là số **chưa VAT**, còn `Tiền BT đã trả cho KH` + `Tiền BT đã trả cho GR` là số thực chi **đã gồm VAT** (dữ liệu 09/2026: 933 HS trả = ước × 1,08; 163 HS không VAT). KPI "Đã chi trả" (`totalDaTra` = trả KH + trả GR) vì vậy được gắn nhãn **"Đã chi trả (gồm VAT)"** và có thể lớn hơn tiền ước/duyệt — không phải lỗi cộng trùng. Không chia ngược 1,08 để quy đổi vì thuế suất không đồng nhất.

## Danh sách Giám định viên

| Mã GĐV | Họ tên |
|--------|--------|
| TUNGHX | Hoàng Xuân Tùng |
| TUYENLM | Lương Minh Tuyến |
| CHINH05 | Nguyễn Hồng Chinh |
| DUYNT | Nguyễn Thế Duy |
| VIETNT05 | Nguyễn Tiến Việt |
| HUONGNV | Nguyễn Văn Hướng |
| SONTT | Trần Thanh Sơn |

## Các tab báo cáo

| Tab | ID | Mô tả |
|-----|----|-------|
| **Tổng quan** | `tab-dashboard` | Tab mặc định. Pane chứa **2 khối con**: `#pane-overview` (KPI · bảng + chart TLBT theo nghiệp vụ · **Doanh thu & TLBT theo kênh khai thác** (Đợt 3) · **Dòng chảy hồ sơ theo tháng** · Infographic theo địa bàn) và `#dash-dyn` (**Phân tích đa chiều** 8 tiêu chí — chính là "Dashboard động" cũ). Giữ id `tab-dashboard` để toàn bộ CSS/JS của phần đa chiều không phải sửa. `renderOverview(s)` ghi vào `#pane-overview`, `renderDashboard(s)` ghi vào `#dash-dyn` |
| **Theo GĐV** | `tab-gdv` | **Bảng "TỔNG HỢP HỒ SƠ TỒN THEO GIÁM ĐỊNH VIÊN (NV XO)" giữ nguyên 100%** (4 hàng header, 18 cột) + **Thẻ điểm chu kỳ GĐV** (đã gộp thêm 3 cột giảm trừ) + bảng HS phát sinh theo tháng + 1 biểu đồ chu kỳ |
| **Tiến độ & Hồ sơ tồn** | `tab-tiendo` | 1. Thời gian TB từng chặng (12 chặng) · 2. Hồ sơ tồn đang nghẽn ở bước nào · 3. **Danh sách hồ sơ tồn** (gộp BC2 + BC3 + Top 50 đứng yên) · 4. **Gara/Showroom** (gộp BC1 doanh thu + BC4 chất lượng phối hợp) · 5. Chất lượng dữ liệu khai báo |
| **Kiểm soát rủi ro** | `tab-ruiro` | 1. Giảm trừ theo lý do · 2. Tổng hợp dấu hiệu cần rà soát · 3. DS hồ sơ cần rà soát (top 100) · 4. DS hồ sơ có giảm trừ (top 60) · 5. Tần suất tổn thất theo xe · 6. Khách hàng ≥5 hồ sơ · 7. Công nợ · 8. Hồ sơ tạm ứng. ⚠️ Mục 2–3 chỉ nêu **dấu hiệu cần rà soát**, không kết luận trục lợi — `.note-box` đầu tab **không được bỏ** |
| **Đề xuất cải tiến CLDV** | `tab-cldv` | 4 **nhóm giải pháp** (11/2026 đổi nhãn "Trục" → "Nhóm giải pháp" cho dễ hiểu): Chất lượng dịch vụ · Hiệu quả & Năng suất · Quan hệ khách hàng · **Liên thông số liệu** (`cldvActionSection(s)`) đặt ngay trước khối Kết luận; hàm `renderCLDV(s)` |
| Tra cứu tiến trình | `tab-search` | **Không có nút tab** (11/2026) — vào từ menu thả xuống (`openSearchTab()`). Nội dung giữ nguyên: tra theo Biển kiểm soát / Số HSBT + lọc gara, GĐV, trạng thái; nút 📥 Tải Excel |
| Tiện ích | `tab-tienich` | **Không có nút tab** — vào từ menu thả xuống (`openTienichTab()`). Bảng liên kết công cụ hỗ trợ giám định, dựng từ mảng `UTIL_LINKS` |

> **Tối ưu hoá 11/2026 — từ 8 nút tab còn 5.** Nguyên tắc: mỗi đối tượng chỉ còn **một** bảng, biểu đồ nào lặp lại đúng nội dung bảng ngay cạnh thì bỏ.
> - **Tab bị xoá**: `tab-overview` (nội dung chuyển vào `#pane-overview` trong tab Tổng quan) và `tab-baocao` (BC1→BC4 gộp thành `garageSection()`, BC2+BC3 gộp vào `tonListSection()`). Hàm `renderBaoCao()` **đã xoá hẳn**.
> - **Bảng đã gộp**: gara (BC1 + BC4 → `garageSection`) · hồ sơ tồn (BC2 + BC3 + Top 50 đứng yên → `tonListSection`) · GĐV (bảng "giảm trừ theo GĐV" ở tab Kiểm soát rủi ro → 3 cột trong `gdvScorecard`).
> - **Biểu đồ: 24 → 13.** Đã bỏ: `chart-bar-main`, `chart-donut-age` (Tổng quan — lặp tab Theo GĐV và tab CLDV); `chart-donut-gdv`, `chart-gdv-tyle`, `chart-gdv-nv`, `chart-gdv-aging`, `chart-gdv-leg` (Theo GĐV — lặp đúng các cột của 2 bảng ngay trên); `chart-stage-stuck` (Tiến độ — lặp bảng 2); `chart-gt-reason`, `chart-gt-gdv` (Rủi ro — lặp bảng 1 và bảng đã gộp vào thẻ điểm GĐV). Đã gộp `chart-flow-ps` + `chart-flow-ton` thành **1 chart** (2 cột + 1 đường tồn lũy kế, cùng trục vì cùng bậc độ lớn). **Tab CLDV giữ nguyên 6 chart.** 11/2026 Đợt 3 thêm `chart-kenh-bt` → 14 chart. **24/11/2026 bỏ tiếp `chart-dash-bucket` + `chart-dash-ratio`** (2 chart đầu khối Phân tích đa chiều — lặp đúng nội dung bảng chi tiết ngay cạnh, người dùng yêu cầu cho gọn) → **12 chart**. Khối Phân tích đa chiều nay chỉ còn **1 `.chart-row`**: donut `chart-dash-money` + bảng chi tiết.
> - **Thứ tự nút tab (5)**: Tổng quan → Theo GĐV → Tiến độ & Hồ sơ tồn → Kiểm soát rủi ro → Đề xuất cải tiến CLDV. Màu `.tabs .tab:nth-child(1..5)` = xanh dương · xanh lá · xám thép · đỏ đậm · hồng đỏ.
> - ⚠️ Khi đổi thứ tự tab vẫn phải sửa đồng bộ **4 chỗ**: (1) thứ tự `<button class="tab">`, (2) thứ tự `<div id="tab-...">` kèm `style="display:none"`, (3) CSS `.tabs .tab:nth-child(n)` (đánh số theo **vị trí**), (4) mảng trong `switchTab()` — mảng này nay là `['dashboard','gdv','tiendo','ruiro','cldv','search','tienich']` và có guard `if(p)` vì 2 pane cuối không có nút.

### Menu thả xuống (trước là menu của "Dashboard động")

- Mở bằng **nút "Xem theo: … ▾"** (`.dash-pick`) nằm trong khối Phân tích đa chiều — đây là lối vào chính. Bấm **lại** nút tab Tổng quan khi đang đứng ở tab đó cũng bật/tắt menu.
- ⚠️ **Không còn tự xổ menu khi vào tab**: nút tab nay mang nghĩa "Tổng quan", xổ menu mỗi lần bấm sẽ gây nhiễu. `switchTab('dashboard')` chỉ toggle menu khi `sameTab`.
- Menu có **9 mục**: 7 tiêu chí nhóm + `🔍 Tra cứu tiến trình` (`openSearchTab()`) + `🧰 Tiện ích` (`openTienichTab()`). Hai mục cuối là **lối vào duy nhất** của 2 tab không có nút.
- ⚠️ `positionDashDD()` **luôn neo vào nút tab Tổng quan** (`dashTabBtn()`), kể cả khi menu được mở bằng nút `.dash-pick` nằm giữa trang — menu là menu của nút tab nên phải xổ ngay dưới nút tab, không bung ra giữa màn hình (yêu cầu của người dùng 11/2026). Làm được vì `.topbar` là `position:sticky` nên nút tab luôn nằm trong khung nhìn dù đã cuộn tới đâu. Đo thật: menu `top = tab.bottom + 8`, lệch ngang 0, đúng cả khi đã cuộn 1.400px và cả khi topbar xuống 2 hàng ở 1.078px. Hàm chỉ còn kẹp chiều dọc cho màn hình quá thấp (ép menu nằm trọn trong khung nhìn).
- Listener click ngoài phải bỏ qua **cả** `.tabs .tab` **và** `.dash-pick`, nếu không cú bấm mở menu sẽ bubble lên và đóng ngay.
- **Chọn mục trong menu là cuộn ngay tới dữ liệu** (11/2026, yêu cầu người dùng): `scrollToBlock(el)` đưa khối lên ngay dưới topbar — `y = el.rect.top + pageYOffset − (topbar.offsetHeight + 10)`; đo `offsetHeight` thật vì ≤1150px topbar xuống 2 hàng (64px → 143px). `setDashDim()` cuộn tới `#dash-dyn` (khối Phân tích đa chiều nằm cuối tab Tổng quan), `openSearchTab()`/`openTienichTab()` cuộn tới pane của chúng (trước đây giữ nguyên vị trí cuộn cũ nên bấm xong tưởng như không có gì xảy ra).
  - ⚠️ **Nhảy thẳng, không `behavior:'smooth'`**: khối vừa render lại, animation cuộn mượt bị cắt giữa đường → dừng lưng chừng (đo thật trong headless: chọn tiêu chí từ y=0 chỉ cuộn được 0–920px thay vì 2.193px). Gọi `jump()` 2 lần (ngay + 1 nhịp `requestAnimationFrame`) để canh lại sau khi chart/bảng co xong.
  - `setDashDim()` có thêm guard: nếu tab Tổng quan đang ẩn thì `switchTab('dashboard', dashTabBtn())` trước khi render, tránh cuộn tới khối đang `display:none`.
  - Đo thật (dữ liệu 10/2026, 2.200 HS): cả 7 tiêu chí đều cho `#dash-dyn` nằm đúng `topbar + 10px` ở 1600 / 1450 / 1100 / 780px (`dynTop` = 74 / 74 / 153 / 148), Tra cứu & Tiện ích về đầu trang, `overflowX = 0`, 0 lỗi JS.
- Trợ lý GĐV (`tlOpenSearch`) tìm nút tab có `onclick` chứa `'search'`; nút đó không còn nên hàm **fallback sang `dashTabBtn()`** — đừng bỏ nhánh fallback này.

### Tiêu chí nhóm của khối Phân tích đa chiều


| dim | Nguồn cột | Giới hạn nhóm (`DASH_TOP_N`) |
|-----|-----------|------------------------------|
| `gdv` | `gdvKey(r)` | không — theo thứ tự `GDV_NAMES` |
| `diaban` | `DIABAN_MAP` | không — theo thứ tự `DIABAN_MAP` |
| `nv` | `Mã nghiệp vụ` → TNDS/VCX/Khác | không |
| `hangxe` | `Hãng xe` (~40 giá trị) | **12** + nhóm "Khác (N hãng xe)" |
| `garage` | `Tên garage (thường gọi)`, fallback `(ĐKKD)` (~118 giá trị) | **12** + nhóm "Khác (N gara/showroom)" |
| `trangthai` | `Trạng thái hồ sơ` (14 giá trị) | không |
| `mucTien` | `Tiền ước/duyệt BT` → 4 khoảng tiền + nhóm chưa ước (`MONEY_BUCKETS`) | không — theo thứ tự `MONEY_BUCKETS` |
| `kenh` | `Mã kênh tổng` (18 giá trị) | **12** + nhóm "Khác (N kênh khai thác)" |

- `MONEY_BUCKETS` (dim `mucTien`, nút "💰 Số tiền bình quân/vụ"): Dưới 10 triệu · Từ 10 đến dưới 50 triệu · Từ 50 đến dưới 100 triệu · Từ 100 triệu trở lên · **Chưa có số tiền ước** (`na:true`, luôn xếp cuối). Ranh giới `min <= m < max`; `moneyBucketKey` đưa mọi giá trị **không > 0** (0, trống, `-`, chữ, số âm) về `MONEY_NA_KEY` **trước khi** dò khoảng — cố ý tách khỏi nhóm "Dưới 10 triệu" để bình quân/vụ của nhóm này không bị kéo tụt. Mỗi bucket có `short` riêng (`< 10 tr`, `10 – <50 tr`, `Chưa ước BT`, …) cho nhãn biểu đồ.
- Bảng `mucTien` có 2 mức bình quân, không được nhầm: hàng **TỔNG** hiển thị `bqEst = t.bt / (t.total − nhóm NA)` (chỉ HS đã có tiền ước), còn `bqTong = t.bt / t.total` chỉ xuất hiện trong dòng chú thích dưới bảng để đối chiếu.
- ⚠️ `computeDashGroups(data, dim, forTable)` được gọi **2 lần** mỗi lần render: `groups` (không truyền `forTable`) cho biểu đồ và `tableGroups` (`forTable=true`) cho bảng. Với `hangxe`/`garage` hai kết quả **cố ý khác nhau**: biểu đồ (nay chỉ còn donut cơ cấu tiền) giữ Top 12 theo số HS, bảng liệt kê mọi nhóm có tổng `Tiền ước/duyệt BT` ≥ `DASH_TABLE_MIN_BT` (100 triệu) xếp theo **tiền giảm dần**, phần dưới ngưỡng gộp hàng "Khác (N … dưới 100 triệu)". Dữ liệu 07/2026: hãng xe 13/40 nhóm (95,5% tiền), gara 27/118 nhóm (80,1% tiền). Chênh lệch này được ghi ở tiêu đề bảng và pill `.dash-info` — đừng "sửa" cho hai bên giống nhau.
- Nhóm gộp cuối có cờ `isKhac` + `khacCount` (số nhóm đã gộp) để bảng tô xám nghiêng và viết chú thích.
- Các dim không có thứ tự chuẩn được xếp theo **số hồ sơ giảm dần**; phần đuôi vượt ngưỡng gộp vào nhóm "Khác" nên **tổng các nhóm luôn khớp tổng toàn cục** (đã có test bất biến).
- `normDimVal(v)`: giá trị rỗng hoặc `"-"` trong file nguồn → gom về nhóm `"(Không xác định)"` (dữ liệu thật có 27 HS trạng thái `-` và 44 HS gara `-`).
- Mỗi nhóm có `label` (đầy đủ, dùng cho bảng) và `short` (cắt ≤22 ký tự, dùng cho nhãn biểu đồ) — tên gara/GĐV dài sẽ phá layout chart nếu dùng `label`.

#### Ba bộ cột của bảng chi tiết Phân tích đa chiều (`dashDetailTable(groups)`)

| dim | Bộ cột |
|-----|--------|
| `mucTien` | Mức tiền · **Tổng số vụ** (`total`) · Tỷ trọng số vụ · **Tổng Tiền ước/duyệt BT** (`bt`) · Tỷ trọng tiền · **Số tiền bình quân/vụ** (`bqVu = bt/total`, format `fmtMBC`, tô màu `--claude`) · Đã GQ · Chưa GQ · Tỷ lệ GQ (`tyLeGQVu`) — kèm hàng `tfoot` **TỔNG** (bình quân tổng tính lại từ tổng tiền / tổng số vụ, không cộng trung bình các nhóm) |
| `hangxe`, `garage` | Hãng xe / Gara-Showroom · **Tổng số vụ phát sinh** (`total`) · **Tổng Tiền ước/duyệt BT** (`bt`, format `fmtMBC`) · **Đã giải quyết** (`daGQ`) · **Chưa giải quyết** (`chuaGQ`) · **Tỷ lệ giải quyết** (`tyLeGQVu`) — kèm hàng `tfoot` **TỔNG** |
| `gdv`, `diaban`, `nv`, `trangthai` | Cần GQ · Đã GQ · Tỷ lệ GQ · Đang tồn · Tồn >45 · Tồn ≥90 · Tỷ lệ giải ngân · Cảnh báo (bộ cột tiến độ tồn, giữ nguyên) |

- ⚠️ Hai công thức tỷ lệ giải quyết **khác mẫu số, cùng tồn tại trong `computeGroupStat`**, không được trộn lẫn:
  - `tyLeGQ = daGQ / tongCanGQ` — chuẩn báo cáo tiến độ (tồn năm trước + PS năm nay), dùng cho GĐV/Địa bàn/Nghiệp vụ/Trạng thái.
  - `tyLeGQVu = daGQ / total`, `chuaGQ = total − daGQ` — góc nhìn theo số vụ phát sinh của nhóm, dùng cho Hãng xe/Gara/Mức tiền để 3 cột Đã GQ + Chưa GQ + Tổng số vụ luôn khớp nhau.
- ~~Với `hangxe`/`garage`, biểu đồ thứ 2 cũng đổi theo (`vuMode`)~~ — **đã bỏ 24/11/2026** cùng 2 chart đầu; cột Đã GQ / Chưa GQ chỉ còn ở bảng chi tiết. Biến `vuMode` không còn trong `renderDashboard` (vẫn còn một biến **trùng tên khác** là `moneyMode` cục bộ trong `computeDashGroups` — đừng nhầm).
- **Chiều cao bảng chi tiết** = đúng chiều cao `.chart-card` cùng hàng, do `syncDashTableHeight()` gán `style.height` sau khi vẽ chart (và khi `switchTab('dashboard')`, khi `resize` — debounce 120ms). Điều kiện để đo đúng: `#tab-dashboard .chart-row{align-items:start}` (không có nó, grid kéo giãn 2 ô bằng nhau → đo ra chiều cao đã bị bảng đẩy lên), và hàm **thoát sớm khi tab đang `display:none`** (offsetHeight = 0 → bảng cao 0px). Màn hình ≤800px `.chart-row` xếp dọc nên dùng chiều cao cố định 420px.
- Bảng cuộn trong `.dash-table-scroll` (`flex:1;min-height:0;overflow:auto`), `thead` sticky top + `tfoot` sticky bottom, scrollbar tuỳ biến màu cyan cùng tông tab. Table trong khung này dùng `border-collapse:separate` — bắt buộc, vì `collapse` làm mất viền ô khi sticky.
- **Không đặt dòng chú thích dưới bảng** (đã gỡ 09/2026) — mỗi dòng chú thích ăn mất ~2 dòng danh mục. Thông tin phụ đặt ở: tiêu đề `<h3>` (ngưỡng lọc của Hãng xe/Gara), pill `.dash-info`, hoặc `title=` tooltip của ô (vd bình quân/vụ tính cả HS chưa ước).
- **Co cột tự động — bảng không có thanh cuộn ngang** (09/2026): `.dash-table-scroll` đặt `overflow-x:hidden`; tiêu đề cột bỏ `white-space:nowrap` (wrap ở khoảng trắng, `overflow-wrap:break-word`) còn ô số giữ `nowrap`. `fitDashCols(card)` đo `table.scrollWidth > scroll.clientWidth` rồi thêm dần các mức **cộng dồn** `dash-narrow-1…5` (giảm padding ngang 10→2px, rồi cỡ chữ) và cuối cùng `dash-wrap-num` + `dash-break-any`. Cỡ chữ của các mức narrow đặt bằng **`em` trên `thead/tbody/tfoot`** để nhân dồn với `.dash-dense-*` (đặt `rem` trên `table`) thay vì ghi đè nhau. ⚠️ Không bao giờ cho bẻ giữa dãy số (`dash-break-any` chỉ áp cho `th` và `td:first-child`) — "927" tách thành "92/7" gây đọc sai. Thứ tự gọi trong `syncDashTableHeight()`: `fitDashCols` → `fitDashRows` → `fitDashCols` (co dòng cũng đổi cỡ chữ nên phải đo lại bề ngang). Đo thật: 1600px → `narrow-1`, 1280px → `narrow-2`, 1024px → mức cuối; `overflowX = 0` ở cả 7 tiêu chí.
- **Menu tiêu chí** (10/2026, sửa 11/2026): khung menu là `#dash-dim-dd.dash-dd > #dash-dim-list.dash-dd-list` đặt **ngay sau `.tabs` trong HTML**, ẩn mặc định (`opacity:0;visibility:hidden`), hiện khi có class `.open`. Mở bằng nút `.dash-pick` trong khối Phân tích đa chiều (xem mục "Menu thả xuống" ở trên) hoặc bấm lại nút tab Tổng quan.
  - **Luồng**: bấm nút tab → `switchTab('dashboard')` → `openDashDD()` (gọi `buildDashDimMenu()` dựng 7 mục từ `DASH_DIM_BTN` + đánh dấu ✓ mục `DASH_DIM`, rồi `positionDashDD()` và gắn `.open`). Chọn 1 mục → `setDashDim()` gọi `closeDashDD()` rồi re-render. Muốn đổi tiêu chí → bấm lại nút tab. Bấm nút tab **khi đang ở tab này và menu đang mở** thì đóng (biến `sameTab` đầu `switchTab` so `style.display` của pane trước khi đổi). Đóng thêm khi: `onmouseleave` trên khung menu, bấm ra ngoài, Esc, hoặc chuyển tab khác.
  - ⚠️ **`position:fixed` + JS định vị** (`positionDashDD()` lấy `getBoundingClientRect()` của `#tab-btn-dashboard`, đặt `top = rect.bottom + 8`, kẹp `left` để không tràn mép phải). Không đặt menu `absolute` bên trong `.tabs`: ở ≤600px `.tabs` có `overflow-x:auto` nên menu sẽ bị cắt. Vì fixed nên phải `positionDashDD()` lại khi `scroll`/`resize` (chỉ khi đang mở).
  - ⚠️ Listener click trên `document` **phải bỏ qua** `e.target.closest('.tabs .tab')` — cú bấm nút tab vừa mở menu sẽ bubble tới listener và đóng ngay trong cùng cú bấm.
  - Nút tab mang `id="tab-btn-dashboard"` + `<span class="tab-caret">▾</span>` (xoay 180° khi có class `.dd-open`). Bọc nút tab trong thẻ khác sẽ **phá màu tab** vì CSS màu gán theo `.tabs .tab:nth-child(n)` — đừng làm, menu đã tách rời khỏi `.tabs`.
  - Nhãn mục lấy từ `DASH_DIM_BTN` (mảng `[dim, nhãn có icon]`, khác `DASH_DIM_LABEL` dùng cho tiêu đề biểu đồ) — thêm tiêu chí mới phải thêm vào mảng này.
  - **Hai dòng cuối droplist không phải tiêu chí nhóm** (11/2026): `buildDashDimMenu()` nối thêm `data-d="search" class="dash-dd-go"` → `openSearchTab()` và `data-d="tienich"` → `openTienichTab()`, cả hai gọi `closeDashDD()` rồi `switchTab(..., dashTabBtn())`. Truyền `dashTabBtn()` vì `switchTab` cần một nút để gắn `.active`. CSS: vàng cam ở `.dash-dd [data-d="search"]`, indigo ở `.dash-dd [data-d="tienich"]`, vạch ngăn ở `.dash-dd-list button.dash-dd-go::before` (đặt ở mục Tra cứu — mục đầu của nhóm lối-vào). Hai mục này không bao giờ có dấu ✓ vì `DASH_DIM` không nhận các giá trị đó.
  - Hàm: `dashDDEl()`, `dashTabBtn()`, `dashDDOpen()`, `buildDashDimMenu()`, `positionDashDD()`, `openDashDD()`, `closeDashDD()`.
  - ⚠️ Kiểm thử bằng Chrome headless: `--virtual-time-budget` tua timer nhưng **không tua CSS transition**, nên ảnh chụp ngay sau khi mở menu ra khung trắng rỗng (opacity vẫn 0) dù DOM đã đủ 7 nút — chèn `*{transition:none!important}` vào bản test trước khi chụp. Biểu đồ vẽ trong `setTimeout` nên kịch bản test phải **chờ giữa các bước**; bấm liên tiếp trong cùng tick làm 2 lần vẽ chồng canvas và chart giữ nhãn cũ.
- **Màu các mục "Xem theo"** (10/2026): mỗi tiêu chí 1 màu riêng gán theo `data-d` (gdv xanh dương · diaban xanh lá · nv tím · hangxe hổ phách · garage cyan · trangthai hồng · mucTien cam · kenh xanh lá chanh), qua 5 biến `--sg-bg1/--sg-bg2` (nền nhạt khi thường), `--sg-fg`, `--sg-d1/--sg-d2` (gradient khi `.active`), `--sg-sh` (bóng). Khai báo trên `.dash-dd [data-d="…"]` nên áp cho mọi mục trong menu. Gán theo `data-d` **không** theo `nth-child` — đổi thứ tự mục không làm lệch màu; thêm tiêu chí mới phải thêm 1 dòng biến tương ứng.
- **Nền bảng chi tiết** (10/2026): `.dash-table-card` lấy đúng `background:#FAFAF5` + bóng của `.chart-card` để 4 khung trong tab cùng tông kem (trước đó là `--surface:#fff` của `.table-card`, lệch tông); `h3` cũng dùng viền dưới `2px var(--claude-light)` như chart-card. Hàng `tfoot` đổi `#f1f5f9` → `#eae4da` cho hợp nền kem.
- **Dòng bảng chi tiết** (10/2026): sọc ngựa vằn `tbody tr:nth-child(even) td` nền `rgba(14,116,144,.055)`; hover `tbody tr:hover td` nền `rgba(34,211,238,.26)` + cột nhãn in đậm kèm vạch cyan `box-shadow:inset 3px 0 0 #0e7490` (dùng box-shadow vì `td:first-child` đã có `border-right` của lưới). ⚠️ Rule hover phải đặt **sau** rule ngựa vằn — hai rule cùng specificity, rule sau mới thắng; cả hai đều mạnh hơn `tr:hover td` chung nên không bị đè.
- **Lưới kẻ bảng chi tiết** (09/2026): kẻ ngang + dọc cho mọi ô, màu `--grid:#cbd5e1`; viền ngoài khung, kẻ phải của **cột nhãn** và đường dưới `thead`/trên `tfoot` dùng `--grid-strong:#94a3b8` (2px, qua `box-shadow:inset` vì ô sticky). Dòng cuối `tbody:last-child` bỏ kẻ dưới để không chồng viền ngoài (bảng GĐV/Địa bàn/Nghiệp vụ/Trạng thái không có hàng TỔNG). Hai biến `--grid`/`--grid-strong` khai báo ngay trên `.dash-table-scroll`.
- **Co dòng tự động**: `fitDashRows(card)` chọn mức thấp nhất trong `DASH_DENSE` = `dash-dense-1/2/3` (giảm dần padding → font → line-height + badge) sao cho khung hiện được `target = min(số dòng, DASH_MIN_ROWS=10)`. Bảng ít mục cũng được co để **hiện đủ không phải cuộn**. `dashVisibleRows()` trừ chiều cao `thead`/`tfoot` sticky và 4px biên khi tính.
- ⚠️ `syncDashTableHeight()` phải chạy **lại sau `document.fonts.ready`** (cờ `_dashFontsBound`): lần render đầu webfont Be Vietnam Pro chưa tải xong, `tr.offsetHeight` đo hụt → chọn nhầm mức co quá nhẹ (đo thật: 11,26 dòng lúc chưa có font nhưng chỉ hiện 9 dòng sau khi font vào).
- **24/11/2026 — khối Phân tích đa chiều chỉ còn donut + bảng.** Hai chart đầu (`chart-dash-bucket` phân bổ bucket ngày / tổng số vụ theo mức tiền, và `chart-dash-ratio` tỷ lệ giải quyết / số vụ Đã–Chưa GQ / bình quân/vụ) **đã xoá** theo yêu cầu người dùng: mọi con số của chúng đã có trong bảng chi tiết ngay cạnh, giữ lại chỉ làm rối mắt. Cùng với đó gỡ 3 biến chỉ phục vụ 2 chart này trong `renderDashboard`: `vuMode`, `moneyMode`, `bucketTitle`, `ratioTitle`; danh sách huỷ chart đầu hàm còn mỗi `chart-dash-money`. `syncDashTableHeight()` vẫn chạy đúng vì nó đo `.chart-card` **cùng `.chart-row`** với bảng — nay là card donut. Muốn khôi phục thì lấy lại từ commit trước 24/11/2026, đừng dựng lại bằng trí nhớ.

## Engine Đợt 1 — đo chu kỳ theo chặng & kiểm soát rủi ro (11/2026)

Hai engine chạy trong `computeStats()` và trả về qua `s.cycle` / `s.risk` (chỉ **thêm trường**, không sửa logic cũ).
Quy ước tồn dùng chung `parseTon` / `isResolved` — **không định nghĩa lại**.

### `computeCycleStats(data)` → `s.cycle`

| Hằng / hàm | Vai trò |
|---|---|
| `STAGE_FLOW` | 11 mốc theo đúng trình tự quy trình, dùng cho `lastStage(r)` (bước dừng cuối). **Không có 3 mốc CNG** — dữ liệu thật chỉ 1–2 HS (0,1%) nên không đo được |
| `STAGE_NEXT` | Bản đồ *bước dừng → việc phải làm tiếp* + class chip khâu phụ trách. Dùng **Make / Check / Validate** (xem mục vai trò bên dưới) |
| `CYCLE_LEGS` | 12 chặng được đo; mỗi chặng có `own` (khâu phụ trách), `cls` (màu chip), `short` (nhãn biểu đồ), `sla` (mốc đọc nhanh **do báo cáo đặt**, không phải thời hạn quy định) |
| `STAT_TIP` · `statLegend()` | Chú thích TB / Trung vị / p90 / Tối đa — gắn `title=` trên `<th>` **và** in thành dòng dưới bảng (tooltip không thấy được trên điện thoại và khi in). Dùng ở cả 3 bảng có cột thời gian: bảng chặng, thẻ điểm GĐV, bảng gara |
| `dayGap(a,b)` · `quantile(sorted,q)` · `statsOf(arr)` | Số ngày giữa 2 mốc · phân vị nội suy · `{n,avg,med,p90,max}` |
| `dataAsOf(data)` | Mốc "dữ liệu đến ngày" = mốc thời gian lớn nhất trong toàn bộ dữ liệu. **Dùng thay `Date.now()`** khi tính số ngày đứng yên — mở lại file cũ mà lấy hôm nay thì số ngày đứng yên bị phóng đại |

- ⚠️ **Giá trị âm bị loại khỏi trung bình** (`neg`): rất nhiều HS có *Ngày giám định chi tiết* sớm hơn *Ngày mở HSBT* (giám định hiện trường trước khi mở hồ sơ — đúng thực tế nghiệp vụ, không phải lỗi dữ liệu). Dữ liệu 10/2026: chặng `Mở HSBT → Giám định chi tiết` có n=1.137 nhưng neg=952, nên cột "HS đo được" hiển thị thêm dòng xám `−952 ngược mốc`. Đừng "sửa" bằng cách lấy trị tuyệt đối.
- `bott` = các chặng xếp theo thời gian TB giảm dần, **đã bỏ chặng do khách hàng** (`own-kh`) để KPI "Nghẽn số 1/2" chỉ nói về phần phòng quản được.
- Tổng số HS trong `flow` **luôn bằng** `tonTotal` (đã kiểm: 166 = 166 với dữ liệu 10/2026) — mỗi HS tồn chỉ thuộc đúng 1 bước dừng.
- `byGdv[].tags` tự gắn nhận định: `w1.med ≥ 10` → "Chậm lấy/trình báo giá"; `w2.med ≥ 15` → "Chậm đốc gara quyết toán"; không dính 2 cái trên mà `cyc.med > 1,5 × trung vị toàn phòng` → "Chu kỳ dài hơn mức chung".

#### Khâu phụ trách — Make / Check / Validate (11/2026, người dùng chốt)

Cột *Khâu phụ trách* (bảng 1) và *Khâu phải xử lý tiếp* (bảng 2) ghi theo **thẩm quyền trong quy trình PTI**, **không còn** "Phòng NV" / "Tài chính" / "GĐV" / "Gara":

| Chặng | Khâu phụ trách |
|---|---|
| Tai nạn → Thông báo TN | Khách hàng |
| Thông báo TN → Mở HSBT | Tiếp nhận |
| Mở HSBT → Giám định chi tiết | **Make** |
| Giám định chi tiết → Trình duyệt giá | **Make** |
| Trình duyệt giá → Check duyệt giá | **Make → Check** |
| Check duyệt giá → Validate duyệt giá | **Check → Validate** |
| Validate duyệt giá → Bảo lãnh | **Validate** |
| Bảo lãnh → Trình ĐNTT | **Make** |
| Trình ĐNTT → Check ĐNTT | **Make → Check** |
| Check ĐNTT → Validate ĐNTT | **Check → Validate** |
| Validate ĐNTT → Duyệt bồi thường | **Validate** |
| Duyệt bồi thường → Trả tiền GR | **Hệ thống tự động** |

- Dạng `A → B` nghĩa là **khoảng chờ bàn giao**: hồ sơ đã trình xong, đang chờ vai trò sau xử lý.
- Hai chặng đầu (Khách hàng, Tiếp nhận) nằm **trước** quy trình Make/Check/Validate nên giữ tên riêng.
- CSS chip: `.own-make` (xanh lá) · `.own-check` (xanh dương) · `.own-val` (tím) · `.own-m2c` / `.own-c2v` (gradient chuyển màu giữa 2 vai trò) · `.own-sys` (xám) · `.own-kh` (tím nhạt). Các lớp cũ `.own-gdv/.own-gara/.own-nv/.own-tc` **vẫn giữ trong CSS và trong `OWN_FILL`** để không vỡ màu nếu còn chỗ nào tham chiếu — đừng xoá khi dọn code mà chưa grep.
- `OWN_FILL` quyết định màu cột của biểu đồ "Thời gian trung bình từng chặng" — thêm vai trò mới phải thêm 1 dòng vào đây, nếu không cột ra màu xám mặc định.

### `computeRiskStats(data)` → `s.risk`

- Giảm trừ: đọc `Tiền giảm trừ BT` + `Lý do giảm trừ BT` (ô rỗng/`-` gom về hằng `RISK_NO_REASON`), cộng thêm `Tiền chia sẻ rủi ro` thành KPI "Tổng tiền tiết kiệm".
- Dấu hiệu rà soát (`flags`, mức `hi`/`md`/`lo`, điểm 3/2/1): TN trước ngày hiệu lực · TN trong 7 ngày đầu hiệu lực · TN trong 8–15 ngày đầu · thanh toán phí **sau** ngày tai nạn · thông báo muộn >5 ngày / 2–5 ngày · **xe có ≥4 vụ trong kỳ**.
- ⚠️ Ngưỡng xe nhiều vụ để ở **≥4** chứ không phải ≥3: với 2.200 HS/10 tháng thì ≥3 vụ trúng 334 HS (phần lớn là xe của khách tổ chức/đội xe — VNPOST, các mã `XO_*`), quá rộng để gọi là dấu hiệu; ≥4 vụ còn 97 HS / 23 xe.
- ⚠️ Tab này **không kết luận trục lợi**. `.note-box` đầu tab nêu rõ đây chỉ là dấu hiệu cần đối chiếu quy tắc, công văn và hồ sơ gốc — giữ nguyên khi chỉnh sửa.

### Thẻ điểm chu kỳ theo GĐV — `gdvScorecard(s)`

Khối phụ **nằm trong tab Theo GĐV** (chèn giữa bảng tồn và hàng chart đầu), không phải tab riêng. Thêm 2 chart `chart-gdv-cycle` + `chart-gdv-leg` vẽ trong `setTimeout` của `renderGdv`. Dùng **trung vị** thay trung bình để 1–2 hồ sơ cá biệt không làm lệch cả GĐV; cột "HS đo" nhỏ nghĩa là GĐV ít hồ sơ đã duyệt BT → **không xếp hạng** khi cột này quá ít.

### CSS kèm theo

`.own-badge` (+ `.own-gdv/gara/nv/kh/tc/sys`) · `.flag-chip` (+ `.flag-hi/md/lo`) · `.legbar>i` · `.note-box` (+ `.info`) · phần nới ô cho `#tab-tiendo`/`#tab-ruiro table.gdv`. Hai tab dùng lại `.gdv-report-wrap` / `table.gdv` / `.chart-card` của các tab cũ nên không phát sinh hệ thống bảng mới.

### Đo thật (dữ liệu 10/2026 — 2.200 HS, as-of 29/10/2026)

Chu kỳ Mở HSBT → Duyệt BT: TB 30,4 · trung vị 22 · p90 63 ngày. Hai chặng nghẽn chiếm **79,1%** tổng thời gian chờ: *Giám định chi tiết → Trình duyệt giá* 13,6 ngày và *Bảo lãnh → Trình ĐNTT* 13,5 ngày; 7 chặng check/validate nội bộ chỉ 0,2–0,9 ngày. Tồn 166 HS: 72 HS dừng ở Giám định chi tiết, 71 HS ở Bảo lãnh. Giảm trừ 302 HS / 1,104 tỷ (193 HS "không thông báo ngay cho PTI"); chia sẻ rủi ro 14 HS / 261,7 tr. Dấu hiệu rà soát: 296 HS, trong đó 80 HS mức cao. Kiểm tra bề ngang: `overflowX = 0` ở 1878 / 1478 / 1258 px và ở 390px (bảng cuộn ngang trong `.gdv-table-outer`).

> Dữ liệu nguồn 10/2026 có **4 dòng tiêu đề trước hàng header** (header nằm ở dòng 6) — khi đọc file bằng script phải dò hàng chứa `Số HSBT` thay vì giả định `header=0`.
> Cột `Tên garage (thường gọi)` có **69 HS ghi "TNDS"** và cột `Trạng thái hồ sơ` có **22 HS ghi "TNDS"** — đây là cách nhập của hệ thống nguồn cho hồ sơ TNDS, không phải lỗi hiển thị của dashboard.

## Engine Đợt 2 — 5 báo cáo bổ sung (11/2026)

Tất cả gắn vào **tab sẵn có**, không thêm nút tab:

| # | Báo cáo | Hàm dựng HTML | Gắn vào |
|---|---------|---------------|---------|
| 3 | Dòng chảy hồ sơ theo tháng (PS · GQ · tồn lũy kế) | `flowSection(s)` ← `computeFlowStats()` | cuối tab **Tổng quan**, ngay trước `renderInfographic()` |
| 11 | Gara: doanh thu + chất lượng phối hợp | `garageSection(s)` ← `computeGarageStats()` | **mục 4** tab Tiến độ & Hồ sơ tồn (11/2026 gộp với BC1) |
| 10 | Tần suất tổn thất theo xe / khách hàng | `repeatSection(s)` ← `computeRepeatStats()` | **mục 6–7** tab Kiểm soát rủi ro |
| 13 | Công nợ & dòng tiền bồi thường | `debtSection(s)` ← `computeDebtStats()` | **mục 8–9** tab Kiểm soát rủi ro |
| 12 | Chất lượng dữ liệu khai báo | `dataQualitySection(s)` ← `computeDataQuality()` | **mục 5** tab Tiến độ & Hồ sơ tồn |

Các hàm `*Section(s)` **trả về chuỗi HTML** để hàm render của tab nối vào `innerHTML`; hàm nào cần biểu đồ thì tự `setTimeout(...,10)` bên trong (timer chạy sau khi chuỗi đã gắn vào DOM nên canvas chắc chắn tồn tại). Engine Đợt 2 **không** nằm trong `computeStats()` — mỗi section tự tính từ `s.rows` để tab nào không mở thì cũng không tốn gì thêm ngoài 1 lượt duyệt mảng.

### Bốn quyết định dễ bị "sửa nhầm" — đừng đảo lại

1. **Tháng đầu dải bị loại khỏi tỷ lệ cả kỳ** (`computeFlowStats`): file xuất của hệ thống cắt theo mốc ngày (bản 10/2026: *Từ ngày 01/01/2026*), nên T11–12/2025 có hồ sơ tồn chuyển sang nhưng **0 hồ sơ duyệt BT** — những hồ sơ duyệt xong trước mốc đã bị cắt. Các tháng trước tháng đầu tiên có `gq>0` được gắn `partial:true`, tô xám, loại khỏi `sumPs/sumGq` và khỏi chart cột PS/GQ. Tính cả vào thì tỷ lệ giải quyết ra **93,2%** (sai, nghe như không theo kịp); loại đúng thì ra **102,8%** trên 10 tháng đủ dữ liệu.
2. **"Gara ảo" TNDS** (`computeGarageStats` + `isRealGara`/`NON_GARA`): 69 hồ sơ ghi `TNDS` vào ô tên gara (hồ sơ TNDS không qua gara). Người dùng đã chốt (11/2026) **giữ dòng này trong bảng** để tổng tiền khớp báo cáo doanh thu cũ → sau khi gộp BC1+BC4, dòng đó vẫn hiện nhưng mang cờ `fake:true`: tô xám nghiêng, **không chấm điểm phối hợp**, 3 cột thời gian để "–", và không tính vào `big`/`slowN`. Đừng xoá dòng, cũng đừng chấm điểm nó.
3. **Công nợ loại hồ sơ hủy và hồ sơ 0 đồng** (`computeDebtStats`): điều kiện thô "đã duyệt BT mà chưa trả tiền" trúng 112 HS, nhưng 83 HS đã hủy + 19 HS tiền ước = 0 → công nợ thật chỉ **10 HS / 40 tr**. Số bị loại hiện ở chân bảng.
4. **Điểm tin cậy dữ liệu chỉ tính lỗi mức bản ghi** (`computeDataQuality`): mục gắn `noScore:true` vẫn hiện trong bảng nhưng không trừ điểm — gồm "nguyên nhân dồn vào 1 giá trị" (91,8% × trọng số 0,6 sẽ kéo điểm xuống 34/100, vô nghĩa vì đó là thói quen khai báo chứ không phải bản ghi lỗi) và "giám định chi tiết trước khi mở HSBT" (43,3%, hợp lệ về nghiệp vụ). Có `noScore` thì điểm ra **93,8/100**.

### Ngưỡng và hằng

- `GQ_MIN_HS = 5` — gara dưới 5 hồ sơ không được gắn nhãn đánh giá (trung vị vô nghĩa). Nhãn: `w2.med ≥ 20` → "Chậm quyết toán", `≥ 12` → "Quyết toán hơi chậm"; `w1.med ≥ 10` → "Chậm gửi báo giá", `≥ 5` → "Báo giá hơi chậm"; không dính gì → "Phối hợp tốt".
- Tần suất xe: bảng liệt kê xe **≥2 vụ**; cột *Khoảng cách 2 vụ gần nhất* tô đỏ khi ≤15 ngày, vàng 16–30 ngày (dấu hiệu tách vụ / trùng hồ sơ). Khách hàng: ngưỡng **≥5 hồ sơ**.
- Công nợ chia 3 dải tuổi nợ: <7 · 7–29 · ≥30 ngày, tính từ Ngày duyệt bồi thường đến `dataAsOf()`.
- **Bảng 8 "Hồ sơ tạm ứng"** (tên cũ "Tạm ứng chưa thu hồi"; sửa 24/11/2026 theo yêu cầu người dùng): 2 cột cuối đổi nghĩa — **Còn phải trả** = `Tiền ước/duyệt BT` − `Tiền TƯ cho KH` (cột AZ − BB), **Đã bồi thường** = `Tiền BT đã trả cho KH` (cột BA, qua `traKhOf`). Hai cột cũ "Đã thu hồi" (`a.thu`) / "Còn phải thu" (`a.con`) **không bị xoá khỏi engine** — `a.con` vẫn là nguồn của `D.advCon` in ở dòng tiêu đề "… còn treo X tr". Mục đích 2 cột mới: hai số **bằng nhau** nghĩa là tạm ứng đã được khấu trừ ngay khi trả nốt, không phải khoản còn treo thật. Dữ liệu 10/2026: cả 3 hồ sơ (đều của HUONGNV, mỗi HS tạm ứng 45 tr) đều bằng nhau — 105/105, 105/105, 30/30 tr. Cờ `a.settled` = `traKH > 0 && conTra − traKH <= 0` (đã trả nốt phần còn lại → tạm ứng coi như đã khấu trừ): hồ sơ `settled` **bỏ tô đỏ** cột Còn phải trả, và dòng tiêu đề nay in **3 số** thay cho câu "còn treo X" cũ — tổng tạm ứng chưa ghi nhận thu hồi (`advCon`) · **thực tế còn treo** (`advTreoN`/`advTreoTien`, cộng `a.con` của hồ sơ chưa settled) · số hồ sơ đã khấu trừ (`advSetN`/`advSetTien`). Dữ liệu 10/2026 ra "3 hồ sơ … (135,0 tr) · thực tế còn treo 0 hồ sơ · 3 hồ sơ đã khấu trừ khi trả nốt (135,0 tr)" — câu cũ "còn treo 135,0 tr" là **sai**, đừng khôi phục. ⚠️ Chỉ trừ `Tiền TƯ cho KH`, **không** cộng `Tiền TƯ cho GR` (đúng cột BB người dùng chỉ định; dữ liệu 10/2026 không có HS nào tạm ứng cho gara).
- **Bộ lọc bảng 8** (24/11/2026, người dùng chốt): hồ sơ vừa có `Tiền BT đã trả cho KH` > 0 (cột BA) **vừa có** `Số ngày tồn` = 0 (cột BK, dùng `isResolved`) thì **không đưa vào bảng** — đã tất toán, tạm ứng nằm trong lần trả cuối. Biến `advDone` đếm số hồ sơ bị loại, in ở dòng tiêu đề và chân bảng. Dữ liệu 10/2026: **cả 3 hồ sơ đều rơi vào diện này** → bảng rỗng; khối **vẫn hiện** (đã bỏ guard `D.adv.length?…:''`) với dòng "Không còn hồ sơ tạm ứng nào phải theo dõi · đã loại 3 hồ sơ tất toán xong" — cố ý, để mục 8 không biến mất khỏi tab mà không rõ lý do. Đã kiểm 2 ca biên bằng hồ sơ giả: BA = 0 & BK > 0 → **giữ**, tô đỏ; BA > 0 & BK > 0 → **giữ**, không tô đỏ. ⚠️ Hồ sơ `Đã hủy` mà BA > 0 và BK = 0 **cũng bị loại** (14H02794 của dữ liệu 10/2026) vì điều kiện đọc theo cột BA, không đọc `Trạng thái hồ sơ`.

### Đo thật (dữ liệu 10/2026 — 2.200 HS)

Dòng chảy: 10 tháng đủ dữ liệu, PS 1.978 / duyệt BT 2.034 = **102,8%**, tồn lũy kế cuối kỳ 166 (đỉnh 290 ở T2/2026). Gara: **74/138 gara có ≥5 HS**, trong đó **27 gara** có dấu hiệu chậm, giữ **5,23 tỷ** tiền ước/duyệt; chậm báo giá nặng nhất là Garage Hoàng Anh Đông Triều (trung vị 47 ngày, 132 HS) và Garage Minh Tân Auto (50 ngày); chậm quyết toán: Garage auto Tiến Đạt 18 ngày, Vinfast Hạ Long 17, Ford Quảng Ninh 16,5. Tần suất: 416 xe ≥2 vụ (962 HS / 8,14 tỷ), 102 xe ≥3, 23 xe ≥4; 39 khách hàng ≥5 HS. Công nợ: 10 HS / 40 tr; tạm ứng 3 HS / 135 tr (từ 24/11/2026 cả 3 bị bộ lọc BA>0 & BK=0 loại khỏi bảng 8, còn **0 HS**); cứu hộ 19 HS / 60 tr. Chất lượng dữ liệu: **93,8/100**.

> **Thanh tab**: giai đoạn 8 nút phải nâng mốc xuống hàng lên 1400px; **từ khi gom còn 5 nút (11/2026) đã trả về 1150px**. Đo thật với 5 nút: 1878px và 1378px không cần co (khoảng hở tiêu đề↔tab 557px / 57px) · 1228px `tt-c1+c2` · ≤1150px topbar wrap. Mọi bề ngang `overflowX = 0`, không phải dùng `tt-scroll`.

## Engine Đợt 3 — doanh thu & TLBT theo kênh khai thác (11/2026)

Khối **"DOANH THU & TỶ LỆ BỒI THƯỜNG THEO KÊNH KHAI THÁC (MÃ KÊNH TỔNG)"** nằm trong tab **Tổng quan**, ngay sau bảng TLBT theo nghiệp vụ (`renderOverview` nối `kenhSection()` vào `#pane-overview`). Không thêm nút tab.

| Thành phần | Vai trò |
|---|---|
| `GDRIVE_KENH_GID='1569610723'` | Sheet **DoanhthuKenh**, **cùng file Google Sheet** với dữ liệu hồ sơ (3 sheet: `XCG05_HSBT` · `ty-le-bt` · `DoanhthuKenh`) |
| `loadKenhSheet()` | **Đường chính: CSV xuất bản** (`/d/e/${GDRIVE_PUB_ID}/pub?gid=…&output=csv`) — giống hệt cơ chế đọc dữ liệu hồ sơ |
| `_kenhCSV(text)` | Tự tách CSV, **giữ mọi ô ở dạng chuỗi gốc**. ⚠ Không đưa qua `XLSX.read`: SheetJS đổi `"1,310,999"` thành số `1310999`, mất dấu hiệu phân biệt ô chữ với ô số |
| `_loadKenhViaGviz()` · `window.ptiKenhCb` | Dự phòng JSONP gviz khi CSV xuất bản không dùng được (mở bằng `file://`, sheet chưa xuất bản) |
| `_applyKenhRecs(recs,src)` | Gom theo `KENH_TONG`; nhận cùng một dạng mảng object (khoá = tên cột viết HOA) từ cả 2 đường đọc |
| `KENH_COL` | Danh sách **tên cột** được chấp nhận — đọc theo header, **không dùng chỉ số cột** |
| `kenhScale(vals)` · `kenhMoney(v,scale)` | Quy ô tiền về **đồng** — xem mục định dạng số bên dưới |
| `kenhRevCheck()` · `KENH_TOL=0.10` | **Cửa kiểm soát**: so tổng `PHI_PS_LK` với doanh thu `XO` của sheet `ty-le-bt` |
| `computeKenhStats()` | Ghép doanh thu với phía bồi thường gom theo `Mã kênh tổng` (dùng chung `parseTon`/`isResolved`/`parseMoney`) |
| `kenhSection()` · `_refreshKenhSection()` | Dựng HTML + 1 chart `chart-kenh-bt`; vẽ lại khi **mỗi** sheet tải xong (`ptiTlbtCb` cũng gọi `_refreshKenhSection()`) |

### ⚠️ Định dạng số của sheet DoanhthuKenh — đã đối chiếu 2.782/2.782 ô (22/11/2026)

Bản xuất của hệ thống ghi số có **dấu phẩy phân nhóm nghìn** (`1,310,999`) — **giống sheet `XCG05_HSBT`**. File Google Sheet đang ở locale VN (phẩy = dấu thập phân) nên mỗi ô rơi vào 1 trong 3 dạng:

| Chuỗi gốc | Google Sheet lưu thành | Đọc về đồng |
|---|---|---|
| `1,310,999` (≥2 nhóm phẩy → vô hiệu) | **CHỮ**, đủ chữ số | bỏ phẩy → `1310999` |
| `437,000` (1 nhóm phẩy) | **SỐ** `437` (bị chia 1.000) | `437 × 1.000` |
| `43,700` | **SỐ** `43,7` | `43,7 × 1.000` |
| `131,101` | **SỐ** `131,101` | `131,101 × 1.000` |

- Ô chữ **không mất chữ số nào** — chỉ **gviz** làm rỗng chúng: gviz gán kiểu cho cả cột theo số đông rồi trả `null` cho mọi ô lệch kiểu. Cột `PHI_PS_LK` có 1.445 ô số / **1.337 ô chữ** → đọc qua gviz mất 1.337 ô = **99,2% doanh thu** (tổng chỉ còn 253 tr/32,46 tỷ = 0,8%). ⚠️ **Đây chính là cái bẫy đã ghi ở mục "Vì sao không chỉ dùng gviz"** — áp dụng y nguyên cho sheet này, nên phải đọc bằng CSV xuất bản.
- Hệ số ×1.000 xác định **theo cả cột** (`kenhScale`), không đoán từng ô: ô số của cột đều < 1.000 → cột đang bị chia 1.000. Sheet về sau được dán lại thành số đồng thật (max ≥ 1.000) thì hệ số **tự về 1**, không phải sửa code.
- Ô 1 nhóm phẩy: ô chữ `131,101` và ô số `131,101` cho **cùng kết quả** (`131×1000 + 101`) nên không cần phân biệt.
- Đã kiểm chéo bằng quan hệ **thuế = 10% phí** trên toàn bộ dòng: 230 ô nguyên mơ hồ (`437` có thể là 437 hay 437.000) **đều** khớp nhánh ×1.000, không ô nào khớp nhánh ×1.
- ✅ **Sheet đã được xuất bản (23/11/2026)**: người dùng chọn *Tệp ▸ Chia sẻ ▸ Xuất bản lên web ▸ **Toàn bộ tài liệu** ▸ CSV*, `GDRIVE_PUB_ID` **không đổi**. Đo lại ngay sau đó: `pub?gid=1569610723` → **200**, 2.782 dòng, `kenhScale` tự nhận **1000**, tổng `PHI_PS_LK` = **33,119 tỷ**, 13 mã kênh — khớp số đã đo trước đó. Không phải sửa dữ liệu trong sheet, cũng không phải sửa code. Trang vẫn phải chạy qua https (GitHub Pages) vì CSV xuất bản không cho `Origin: null` của `file://`.

### ⚠️ Cửa kiểm soát doanh thu — đừng bỏ

`showRev = rv.ok` (lệch ≤ `KENH_TOL` = 10% so với doanh thu XO của `ty-le-bt`). Không khớp → **cột Doanh thu / Tỷ trọng DT / TLBT để trống** + `.note-box` nêu đúng nguyên nhân theo đường đọc (`src==='gviz'` → hướng dẫn xuất bản sheet; `src==='pub'` → nhắc kiểm cột và kỳ số liệu); phía bồi thường vẫn hiện đủ. Không đối chiếu được (sheet `ty-le-bt` chưa tải) cũng **ẩn** — hiện số sai còn tệ hơn để trống. Lý do phải có cửa này: **cùng một sheet**, đọc qua gviz ra **0,8%** còn đọc qua CSV xuất bản ra **102,0%**.

Ngưỡng 10% (không phải 5%): `PHI_PS_LK` = 33,119 tỷ còn "D.thu năm nghiệp vụ" của `ty-le-bt` = 32,464 tỷ → lệch **+2,0%** do khác mốc ghi nhận (`PHI_PS` 33,119 · `PHI_BH` 33,223 · `PHI_TT` 32,147 tỷ), không phải lỗi đọc.

### Ghi chú nội dung

- Sheet kênh là **dòng chi tiết cán bộ × mã nghiệp vụ** (2.782 dòng, 13 mã `KENH_TONG`), không phải bảng đã tổng hợp → báo cáo **tự gom về mã kênh tổng**; cột tiền dùng `PHI_PS_LK` (phí phát sinh lũy kế, **chưa VAT**). Dòng thiếu mã kênh vẫn vào nhóm `(Không xác định)` để tổng khớp; chỉ bỏ dòng rỗng hoàn toàn của CSV.
- Bảng 12 cột: Mã kênh tổng · Tên kênh · Doanh thu phí LK · Tỷ trọng DT · Số HSBT · Tiền ước/duyệt BT · Tỷ trọng BT · TLBT % · Đã GQ · Đang tồn · Tồn ≥90 (+ `tfoot` TỔNG). Chart: có doanh thu → cột doanh thu + đường TLBT (`_buildDualNvBtChart`); chưa có → cột tiền BT + số HS (`buildBar`).
- ⚠️ **TLBT ở bảng này = Tiền ước/duyệt BT ÷ Doanh thu phí lũy kế** → **số ước tính nội bộ**, cố ý khác TLBT chính thức của `ty-le-bt` (tính theo bồi thường thực trả năm nghiệp vụ): 60,3% so với 49,18%. Đã ghi rõ ở `gdv-report-footer`.
- ⚠️ `KENH_NAME` **chỉ điền tên cho mã đã đối chiếu được** bằng cột `Mã kênh bán` của chính dữ liệu hồ sơ (NHK = RBSEA/RBVIB/RBVCB/OCVIETIN… → ngân hàng; SH = SH.TOYOTA/SH.FORD… → showroom). Mã chưa rõ (`DKPTI`, `HHI`, `KHCL`, `PTI.KTTT`, `CILC.H`, `KHNN`, `MWG`, mã số) **giữ nguyên mã, không đoán tên**; `title=` của ô mã liệt kê các mã kênh bán thuộc kênh đó.

### Đo thật (dữ liệu 19/11/2026 — 2.334 HS · sheet kênh 2.782 dòng)

Đọc qua CSV xuất bản: doanh thu **33,119 tỷ** (102,0% mốc `ty-le-bt`), tiền ước/duyệt BT **19,968 tỷ**, TLBT chung **60,3%**, 20 mã kênh (hợp của 2 nguồn). Theo kênh: 00.GARAGE DT 10,268 tỷ / BT 4,889 tỷ / **TLBT 47,6%** · SH 8,219 / 5,562 / **67,7%** · VNPOST 6,688 / 2,922 / **43,7%** · NHK 3,705 / 2,768 / **74,7%** · DKPTI 1,751 / 1,110 / **63,4%** · KHONG_KENH_KT 1,600 / 1,698 / **106,1%**. Đọc qua gviz: 0,8% → hiện cảnh báo, cột doanh thu/TLBT để trống, phía bồi thường vẫn đủ 2.334 HS. Cả 2 đường: **0 lỗi JS**, `pageOverflowX = 0` ở 1600/1280/1100/780/390px.

> **Cách kiểm lại ô thật khi nghi số sai** (không cần trình duyệt): `curl -L "https://docs.google.com/spreadsheets/d/<ID>/export?format=xlsx" -o book.xlsx` rồi đọc `xl/worksheets/sheet3.xml` + `xl/sharedStrings.xml` bằng `zipfile` — ô `t="s"` là **chữ**, ô không có `t` là **số**. Đây là cách duy nhất thấy được ô nào là chữ, ô nào là số; `gviz` và CSV đều đã qua một lớp xử lý. `/export` chạy được bằng `curl` nhưng **không** dùng được trong trình duyệt (redirect 307 thiếu header CORS).

## Nhóm giải pháp 4 tab CLDV — liên thông số liệu từ tab Tiến độ & tab Kiểm soát rủi ro (11/2026)

`cldvActionSection(s)` trả về chuỗi HTML, được `renderCLDV` nối vào `innerHTML` **ngay trước khối "Kết luận & Mục tiêu kế hoạch kỳ tới"**. Mục đích: 3 nhóm đầu đề xuất theo hướng chung, nhóm 4 lấy đúng kết quả **đã đo** ở 2 tab kia và gắn mỗi con số với *một việc làm — khâu chủ trì (Make/Check/Validate) — chỉ tiêu đo — kỳ hạn*.

| Khối | Nguồn dữ liệu | Nội dung |
|---|---|---|
| KPI (8 ô) | `s.cycle`, `s.risk`, `computeGarageStats`, `computeDataQuality`, `computeDebtStats` | Nghẽn số 1/2 · HS đứng yên >30 ngày · gara chậm/tổng gara ≥5 HS · HS cần rà soát ngay · tiền tiết kiệm · công nợ · điểm tin cậy dữ liệu |
| 4.1 Điểm nghẽn theo chặng | `s.cycle.legs` (top 6 theo TB) | Cột TB/Trung vị/Mốc nội bộ/Vượt mốc/% chu kỳ + **Sáng kiến** lấy từ `CLDV_LEG_FIX` (khoá = `lab` của chặng) + chỉ tiêu `Math.max(sla, avg×0,7)` |
| 4.2 Tồn dừng ở bước nào | `s.cycle.flow` (top 6 theo số HS) | Kèm chip "Khâu phải xử lý tiếp" (`STAGE_NEXT`) + **Việc phải chốt** từ `CLDV_FLOW_FIX` (khoá = tên bước dừng) |
| 4.3 Gara cần siết cam kết | `computeGarageStats().big` lọc `slow`, top 10 theo tiền | Hành động dựng bởi `cldvGaraAct(g)` — tự chèn trung vị báo giá/quyết toán và số HS tồn của chính gara đó nên mỗi dòng khác nhau |
| 4.4 Sáng kiến rủi ro – công nợ – dữ liệu | `s.risk`, `computeDebtStats`, `computeRepeatStats`, `computeDataQuality` | Mảng `riskIdeas` dựng động: lý do giảm trừ lớn nhất · giảm trừ không ghi lý do · HS dấu hiệu mức cao · công nợ · tạm ứng · xe ≥4 vụ/khách ≥5 HS · 2 lỗi nhập liệu nặng nhất (bỏ mục `noScore`) |

- **Không thêm biểu đồ** (tab CLDV vẫn đúng 6 chart): mọi số ở đây đã có biểu đồ tương ứng ở 2 tab nguồn, vẽ lại là lặp đúng theo nguyên tắc tối ưu 11/2026.
- Dùng lại hệ bảng `.gdv-report-wrap` / `table.gdv` / `.own-badge` / `.flag-chip` của 2 tab kia; CSS nới ô ở dòng `#tab-tiendo … ,#tab-cldv table.gdv` + 2 rule riêng cho ô `td.left` (câu dài: `white-space:normal`, `vertical-align:top`).
- Thêm tiêu chí mới cho một chặng / một bước dừng thì **phải thêm khoá vào `CLDV_LEG_FIX` / `CLDV_FLOW_FIX`**, nếu không ô sáng kiến rơi về câu mặc định.
- Khối **Kết luận** cũng được nối thêm 4 mục mốc lấy từ `s.cycle`/`s.risk` (biến `CY`, `RK`, `bn1`, `bn1Tgt`, `cycMed`, `cycTgt`) — cố ý **chỉ đọc 2 trường đã tính sẵn trong `computeStats()`**, không gọi lại engine để khỏi thêm lượt duyệt mảng.
- Đo thật (dữ liệu 10/2026 — 2.200 HS, as-of 29/10/2026): 4 bảng = 6 + 6 + 10 + 8 dòng; KPI ra đúng số của 2 tab nguồn (nghẽn 13,6 / 13,5 ngày · 50 HS đứng yên >30 ngày · 27/74 gara chậm giữ 5,231 tỷ · 80/296 HS dấu hiệu · công nợ 10 HS / 39,9 tr · 93,8/100). `pageOverflowX = 0` ở 1600 / 1280 px và ở 390px (bảng cuộn trong `.gdv-table-outer`), 0 lỗi JS.

## Trợ lý ảo GĐV (popup "💬 Trợ lý GĐV")

Popup hỏi–đáp quy trình giám định – bồi thường XCG của PTI và tra hồ sơ trong dữ liệu đang tải. **Chạy offline, không gọi AI/Internet** (phương án A: tra cứu bộ tri thức soạn sẵn). Không phải tab — **không** thêm vào `switchTab()`/thanh tab.

- **Vị trí DOM**: `#tl-fab` (nút nổi) + `#tl-panel` (khung chat) nằm **bên trong `#dashboard`**, sau `<footer>` → chỉ hiện khi dashboard hiện (ẩn ở màn đăng nhập/upload/loading). `position:fixed`, z-index 1000/1001 (> topbar 100 > thanh tab 90). Màn hình ≤600px: panel phủ toàn màn hình, `#dashboard.tl-open` ẩn nút nổi.
- **Logo robot AI** (09/2026): SVG vẽ 1 lần trong `<symbol id="tl-bot-art">` (ngay sau `#tl-fab`), dùng lại bằng `<svg class="tl-bot"><use href="#tl-bot-art"/></svg>` ở nút nổi và header popup. Mắt chớp + đèn ăng-ten đổi màu bằng SMIL `<animate>` (chạy được qua `<use>`, CSS thì không); nút nổi nhún nhẹ `tlBob`, rung `tlWiggle` khi hover, tắt khi `prefers-reduced-motion`.
- **Điện thoại ≤600px** (09/2026): panel phủ toàn màn theo `visualViewport` (`tlFitViewport()` gán `--tl-vh`/`--tl-top` → khung co lại khi bàn phím bật, ô nhập không bị che); ô nhập `font-size:16px` (nhỏ hơn thì iOS tự phóng to trang); không tự focus ô nhập khi mở. Bảng trong câu trả lời: `tlFitTables()` chuyển sang **dạng thẻ** (`table.tl-stack`, mỗi dòng 1 thẻ, mỗi ô "nhãn cột : giá trị" qua `data-label` do `tlLabelTable()` gắn + bọc `span.tl-cv`) khi bảng tràn ngang hoặc có ≥3 cột và ô chữ >45 ký tự; bảng 2 cột ngắn giữ dạng bảng. Chạy lại khi resize/xoay màn hình. Máy tính không đổi.
- **Hỏi bằng giọng nói** (09/2026): nút micro `#tl-mic` giữa ô nhập và nút Gửi, dùng Web Speech API (`SpeechRecognition || webkitSpeechRecognition`, `lang='vi-VN'`, `interimResults`). Trình duyệt không hỗ trợ (Firefox) → nút ẩn. Đang nghe: nút đỏ nhấp nháy, chữ tạm hiện trong ô nhập; nói xong tự gửi qua `tlAsk()`. `tlVoiceNorm()` ghép biển số đọc rời ("14 A 123 chấm 45" → `14A12345`) và bỏ dấu câu cuối. Lỗi quyền micro/mạng/không có micro → tin nhắn hướng dẫn (không lưu vào lịch sử). Đóng popup/Esc khi đang nghe → `tlVoiceCancel()` huỷ, không gửi. ⚠️ Chrome/Edge gửi **âm thanh** lên máy chủ nhận dạng của trình duyệt (cần Internet) — phần trả lời vẫn tra offline; trang phải chạy qua https (GitHub Pages) hoặc file:// mới xin được quyền micro. Kiểm thử headless phải giả lập **cả** `SpeechRecognition` lẫn `webkitSpeechRecognition` (Chrome có sẵn bản không tiền tố).
- **Bộ tri thức**: `assets/tro-ly-gdv-kb.js`, nạp bằng `<script src>` ngay trước script chính. Khai báo `TL_META`, `TL_GROUPS` (13 nhóm — thêm `phaply` "Khung pháp lý" và `tinhhuong` "Tình huống thực tế" 09/2026), `TL_FORMS` (73 biểu mẫu BM.PTI.XCG.20.xx + NVXE039.01.xx), `TL_SLA_B4` (thời hạn Bước 4 theo số tiền), `TL_STATUS` (trạng thái phần mềm → bước quy trình + việc cần làm), `TL_KB` (147 mục). Thiếu file → trợ lý vẫn tra được hồ sơ.
- **Nguồn nội dung (ưu tiên quy trình PTI)** — toàn bộ nằm trên **Google Drive** (xem mục "Kho tri thức trên Google Drive" bên dưới; trước 11/2026 là `docs/Kho tri thức/`): QT giải quyết YCBT PTI.XCG.20 (QĐ 20/12/2024) + phụ lục PL.PTI.XCG.20.01–20.18; Quy tắc VCX ô tô QĐ 109/QĐ-PTI ngày 23/09/2025 (hiệu lực 14/10/2025) + HD triển khai T10/2025; Quy tắc QĐ 110/QĐ-PTI ngày 23/09/2025 (trách nhiệm chủ xe/lái xe với phụ xe, người ngồi trên xe) + PL01 bảng tỷ lệ trả tiền thiệt hại về người; Quy trình thu hồi & thanh lý tài sản sau bồi thường (01/07/2025); CV 4406 (cam kết CLDV), CV 3653 (ảnh cấp đơn), CV 1959/PC-PTI 22/05/2025 (thanh toán phí), CV 5884; sổ tay nội bộ "Tiện ích tra cứu CV" (mở logic iLead, khuyến nghị KTNB, sai phạm cấp đơn, mốc 19/07/2024, ấn chỉ & quy tắc áp dụng, chứng từ thiệt hại tài sản NT3); cam kết dịch vụ PTI Quảng Ninh 2019 (bản gốc do Ban XCG cung cấp, **không** lưu trong kho — người dùng chọn giữ mục 28/09/2026); CV 3770/PTI-BHXCG 11/10/2024 + CV 25/PTI-BHXCG 25/11/2025 (phí tái tục VCX, mục `qt-taituc`); **văn bản pháp luật**: Luật KDBH 08/2022 + Luật 139/2025 (dùng bản hợp nhất 31/VBHN-VPQH), NĐ 67/2023, NĐ 220/2026, NĐ 46/2023 + TT 67/2023/TT-BTC (mục `pl-tt67`), BLDS 91/2015; **sổ tay tham khảo** (gắn `v:0`): Bộ Q&A 300 tình huống + "Nhóm tình huống thực tế GĐBT bảo hiểm xe cơ giới" → nhóm `tinhhuong` 15 mục. Văn bản ban hành trước QT PTI.XCG.20 gắn `v:0`.
- ⚠️ **Chỉ dùng tài liệu còn trong kho Drive.** (Tới 11/2026 câu này đọc là "còn trong `docs/`".) 09/2026 người dùng thay `docs/Thi dinh ky/` bằng `docs/Kho tri thức/` → đã gỡ 6 mục trích từ công văn không còn bản gốc (`cl-app` CV 2186, `cl-hdkh` CV 2479, `ct-xacnhan5tr` CV 87, `gd-nganhang` CV 5602, `td-nguoi-xacminh` CV 756, `td-xemay` CV 2299) và dọn mọi tham chiếu `r:` tới chúng; `gd-thanhtoanphi` được **dựng lại từ CV 1959** thay cho CV 1933 cũ. Trước đó (cùng 09/2026) đã gỡ `dg-mautrinh`, `sb-baocao` khi xoá `docs/Tien ich/` và `Mau bieu-GD/` — riêng `sb-mologic` nay **khôi phục** vì nội dung mở logic đã có lại trong "Tiện ích tra cứu CV". Không thêm lại mục nào khi chưa có tài liệu gốc. Bản gốc (phần lớn là PDF scan) nằm trên **Drive riêng tư** — **không** có trên trang web.
- ⚠️ **Đã gỡ 6 mục mất nguồn (23/11/2026, người dùng chốt "xoá, không cần nữa")**: sổ tay nội bộ **"Tiện ích tra cứu CV"** không còn ở máy **và** không có trên Drive, nên 6 mục trích từ nó đã bị xoá khỏi `TL_KB` cùng mọi tham chiếu `r:` và 9 câu kiểm thử tương ứng trong `PHRASINGS` của `tools/test-search.js`: `tn-anchi-quytac` (ấn chỉ & quy tắc áp dụng), `sb-mologic` (mở logic iLead + bảng phân quyền), `sb-ktnb` (khuyến nghị Kiểm toán nội bộ), `sb-saipham` (báo cáo sai phạm cấp đơn), `sb-19072024` (mốc 19/07/2024), `td-taisan-ct` (chứng từ thiệt hại tài sản NT3). **Không thêm lại** khi chưa có bản gốc. `tools/check-kb.js` nay có mảng **`DEAD_SRC`** chặn mọi nguồn `s:` viện dẫn lại sổ tay này — tài liệu nào bị gỡ khỏi kho thì thêm 1 dòng vào mảng đó.
- **Rà kho 28/09/2026** (người dùng cắt dữ liệu trùng: 443 → 261 file, 563 → 351 MB): mọi nguồn `s:` còn tài liệu gốc, **không mục nào phải gỡ**. NĐ 67/2023 vốn đã trích gián tiếp qua `PL.PTI.XCG.20.12` (kho chỉ có CV 3351B hướng dẫn NĐ 67, không có bản NĐ gốc) — đúng như trước, không phải mất nguồn. Nhóm `nt3` đã dựng theo **PL.XCG.20.04 bản "mới"** (biểu mẫu NVXE039.01.xx, Bước 6 đánh giá năng lực tài chính NT3, Bước 7 khởi kiện) — đã đối chiếu lại bản gốc, khớp. Thêm `qt-taituc` (nhóm `quytac`) và `pl-tt67` (nhóm `phaply`). **Còn chưa đưa vào** (người dùng chưa chọn): CV 3351B, CV 3381 (CLDV kênh Banca), Quy tắc bảo hành mở rộng MR, Bảng so sánh QT GĐBT cũ ↔ QT giải quyết YCBT mới, HD GĐBT TNDS hàng hoá (pptx), và `TL_FORMS` chưa đối chiếu với thư mục `Quy trinh XCG -bieu mau 2026/`.
- **Lưu ý pháp lý đã đối chiếu bản gốc (đừng suy diễn lại)**: NĐ 220/2026/NĐ-CP (22/06/2026, hiệu lực 01/07/2026) **chỉ sửa phần bảo hiểm công trình xây dựng** của NĐ 67/2023 (Điều 32–38, Phụ lục III, Mẫu số 3 PL X); với XCG nó chỉ **đổi thuật ngữ "người thứ ba" → "bên thứ ba"** (Điều 9) — mức trách nhiệm, phạm vi, loại trừ, tạm ứng, hồ sơ TNDS **không đổi**. Luật KDBH **không có "thời hiệu 3 năm"**; mốc trong Luật là **Điều 30 – nộp hồ sơ YCBT 01 năm** và **Điều 31 – trả tiền 15 ngày nếu HĐ không thoả thuận, chậm trả phải trả lãi**.
- **Mục TL_KB**: `id`, `g` (nhóm), `q` (câu hỏi chuẩn), `k` (từ khoá/cụm đồng nghĩa), `a` (trả lời — định dạng rút gọn: `- ` gạch đầu dòng, `|a|b|` bảng, `!! ` khung lưu ý, `**đậm**`), `s` (nguồn — **bắt buộc**), `r` (id liên quan), `v:0` (nội dung tham khảo), `x` ('forms' | 'status' | 'setgdv' → engine nối thêm bảng động).
- ⚠️ **Không đưa họ tên, SĐT, email cán bộ vào bộ tri thức** — file nằm cùng trang web (GitHub Pages công khai), ai mở trang cũng đọc được (đăng nhập của trang chỉ là kiểm tra phía trình duyệt).
- **Thứ tự trả lời** trong `tlAsk(q)`: `tlSmallTalk` (chào/cảm ơn) → `tlAnswerData` (biển số/HSBT/tồn) → `tlAnswerForm` (mã mẫu) → `tlAnswerKB`.
- **Tìm kiếm** (`tlSearch`): `tlNorm` = `normText` bỏ dấu + chỉ giữ chữ/số (giữ dấu chấm giữa 2 chữ số, vd `20.35`) + gộp **"tổn thất" → `tonthat`** (tránh lẫn với "tồn" sau khi bỏ dấu); bỏ stopword `TL_STOP`; mở rộng viết tắt `TL_ABBR` (tnds, vcx, nt3, gdv, bt…). Điểm = cụm từ khoá khớp nguyên cụm (`2 + 1.5×số từ`; cụm nằm trọn trong cụm dài hơn cũng khớp thì chỉ tính cụm dài) + từ đơn có trọng số **IDF** (khớp câu hỏi ×1.2, từ khoá ×0.8, nội dung ×0.2); khớp đúng câu hỏi chuẩn +100. Dưới `TL_MIN_SCORE = 4.5` → trả "chưa tìm thấy" + gợi ý, **không tự suy đoán quy định**.
- **Tra dữ liệu** (`tlAnswerData`, đọc `RAW`): biển số (`\d{2}[A-Z]{1,2}\d{4,6}` sau `normPlate`, chấp nhận "14A 123.45"); số HSBT dạng đầy đủ `…/BT/…` hoặc 5–7 số **bắt đầu bằng 0** (số ngắn không khớp HSBT → thử như một phần biển số); "tra/tìm <mảnh số>" → khớp một phần cả hai. Thẻ hồ sơ hiển thị mốc ngày gần nhất + bước quy trình theo `TL_STATUS` + thời hạn Bước 4 theo `Tiền ước/duyệt BT`. "Hồ sơ tồn của <mã/tên GĐV>", "hồ sơ của tôi", "tồn ≥90 ngày" (`tlThreshold`: mốc 90 luôn hiểu ≥90), "tổng quan tồn". Câu có "tính thế nào/là gì/cách tính…" → chuyển sang bộ tri thức. Quy ước tồn **giống `computeStats()`** (`parseTon`, `isResolved`, >45, ≥90) — đã đối chiếu khớp 1.367 / 194 / 38 / 9 với dữ liệu 07/2026.
- **CSS**: khối `.tl-*` cuối `<style>`. Bảng trong khung chat phải ghi đè CSS chung `th{text-transform:uppercase;white-space:nowrap}` và `tr:last-child td{border-bottom:none}`; `tlPush` tự bọc `<table>` trong `.tl-tbl` (cuộn ngang, không tràn popup). Ô bảng dùng `overflow-wrap:normal` (tránh ngắt giữa mã GĐV "TUNG|HX"); đoạn văn dùng `anywhere` (số HSBT dài).
- **Sự kiện**: nút trong khung chat dùng `data-tl-act` (kb | grp | ask | rec | open | setgdv) + 1 listener uỷ quyền trên document — không nhúng nội dung người dùng vào `onclick`. Esc đóng popup. "Mở ở tab Tra cứu tiến trình" gọi `switchTab('search')` rồi điền ô và `doSearchHSBT()`.
- **Bổ sung nội dung**: thêm mục vào `TL_KB` (id không trùng, **bắt buộc ghi `s`**), thêm từ khoá theo cách GĐV hay hỏi, rồi chạy 3 script trong **`tools/`** (xem `tools/README.md` — quy trình 5 bước):
  ```
  node tools/check-kb.js        # toàn vẹn: id, nhóm, r:, thiếu s/k, lộ email/SĐT, nguồn chết
  node tools/test-search.js     # 147/147 câu chuẩn + 27 câu diễn đạt + mục PTI thắng mục v:0
  node tools/compare-search.js  # so trước/sau (lấy bản cũ từ git) — chỉ đọc mục ③
  ```
  Mốc 23/11/2026: **147/147** câu chuẩn, 27/27 câu diễn đạt, **0 hồi quy**. Nguyên tắc: mục mới (nhất là nhóm `tinhhuong` `v:0`) lấn mục quy trình PTI thì **thêm từ khoá cho mục PTI**, đừng hạ mục mới. Hai trường hợp đã biết và **không phải hồi quy** (đã ghi trong `KNOWN` của `test-search.js`), đừng "sửa" bằng cách chỉnh mục cũ: "xe ngập nước giám định thế nào" → `tn-dacbiet` thắng `gd-ngapnuoc` (vẫn hiện `gd-ngapnuoc` ở gợi ý liên quan), và "công thức nấu phở" trúng `dg-congthuc` 8.0 vì từ khoá "công thức". `tools/kb-env.js` cắt engine khỏi `index.html` theo 2 mốc `function normText(`→`function normPlate(` và `const TL_LS_LOG=`→`function tlAsk(` — **đổi tên các hàm/hằng đó phải sửa 2 mốc này**. `tlBuildIndex` mở rộng viết tắt (`tlExpand`) cả cho câu hỏi chuẩn `_qn` — thiếu bước này, câu chuẩn có "TNDS" không bao giờ khớp nguyên văn (+100) vì câu người dùng đã được mở rộng (trước đây 3 mục `td-loaitru`, `td-nguyentac`, `td-hoso` bị `td-mtn` lấn). Giao diện điện thoại kiểm bằng Chrome headless chụp trang nhúng trong iframe 390px (Chrome desktop có độ rộng cửa sổ tối thiểu nên `--window-size=390` không mô phỏng đúng).

### Kho tri thức trên Google Drive (11/2026 — nguồn tài liệu gốc)

Từ 22/11/2026, **tài liệu gốc của bộ tri thức nằm trên Google Drive**, không còn ở `docs/Kho tri thức/`:

| | |
|---|---|
| Thư mục | **Kho tri thức - Trợ lý GDV** |
| Folder ID | `1KdSBw_qWJeX2dl97ARJucfFQkMRs9WaQ` |
| Chủ sở hữu | `chinhnh.pti@gmail.com` — **riêng tư**, không chia sẻ công khai |
| Quyền của Claude | **chỉ đọc** (`canEdit:false`) — không ghi/xoá được |
| Đọc bằng | connector `claude.ai Google Drive` (`search_files`, `read_file_content`) |

⚠️ **Đây là nguồn dựng KB, KHÔNG phải nguồn chạy lúc mở trang.** Trang web vẫn chỉ nạp `assets/tro-ly-gdv-kb.js`, chạy offline, không gọi mạng. Lý do không cho trình duyệt đọc Drive trực tiếp:
1. Trợ lý là máy **so khớp từ khoá** trên các mục `q`/`k`/`a`/`s` soạn tay — không "hiểu" được text thô, và trong trình duyệt không có OCR.
2. Liệt kê thư mục Drive cần API key/OAuth + CORS; nhúng key vào trang công khai là lộ key, mà kho lại riêng tư.
3. ~340 MB PDF mỗi lần mở trang.

**Quy trình cập nhật KB**: người dùng thả tài liệu vào thư mục Drive → báo Claude → Claude đọc qua connector, thêm/sửa mục `TL_KB` (bắt buộc ghi `s`) → chạy 3 script `tools/` → commit + push lên `main`.

**Cạm bẫy của connector Drive (đã đo thật 22/11/2026 — đừng mất công phát hiện lại):**
- **Trần 5 kết quả/lượt**, bất kể `pageSize` (thử tới 1000). Liệt kê ~190 file cần ~40 lượt tuần tự → **đừng vét cạn**; hãy truy vấn đúng tài liệu cần bằng `title contains` / `fullText contains`.
- Tài liệu connector cảnh báo *"Duplicate results can occur across page boundaries"* → bản diff dựng từ phân trang **không đủ tin cậy** để làm căn cứ xoá bản gốc.
- **`in ancestors` trả cả file trong thùng rác.** Phải thêm `trashed = false` thủ công (kho này có thư mục "Thẩm quyền duyệt HS" đang bị trash kèm `2026.pdf`).
- **Drive chuyển định dạng khi tải lên và cắt phần mở rộng**: `.docx`→Google Doc, `.xlsx`→Sheet, `.pptx`→Slides. So khớp tên theo `*.docx` sẽ không ra gì. PDF thì giữ nguyên `.pdf`.
- ✅ **Drive tự OCR được PDF scan** — `read_file_content` trả về text dùng được của công văn scan (đã thử CV-4406: lấy đúng mốc "Báo giá dưới 20 triệu tối đa 8 giờ"). Đây là năng lực **mới** so với đọc file ở máy. Nhưng ô tích trong bảng ra nhiễu (`√` → `ฟ`, `نحل`), nên **bảng ma trận vẫn phải đối chiếu bằng mắt**, đừng tin OCR cho phần đánh dấu.

**Đối chiếu Drive ↔ máy (22/11/2026)**: toàn bộ **153/153** nguồn `s:` của KB (bản 153 mục lúc đó) đều còn tài liệu gốc trên Drive — 18 phụ lục `PL.XCG.20.01–20.18` (cả 2 bản `20.04` cũ/mới), QĐ ban hành PTI.XCG.20, QĐ 109 + QĐ 110, CV 1959/2896/3351B/3381/3653/3770/4406/5884 + CV 25, NĐ 46/2023, NĐ 220/2026, TT 67/2023, Luật KDBH 08/2022, Luật 139/2025, VBHN 31/VPQH, BLDS 91/2015, Quy trình thu hồi & thanh lý, Bộ Q&A 300, Nhóm tình huống thực tế, Quy tắc bảo hành MR, Bảng so sánh QT cũ↔mới, HD GĐBT TNDS HH, bộ biểu mẫu BM.PTI.XCG + biểu mẫu 2026. **Ngoại lệ duy nhất** là sổ tay "Tiện ích tra cứu CV" — 6 mục trích từ nó **đã gỡ ngày 23/11/2026**, KB còn **147 mục** và 147/147 nguồn đều có bản gốc trên Drive.

## Cấu hình Địa bàn (dùng cho khối Infographic trong tab Tổng quan)

```js
const DIABAN_MAP = {
  'Móng Cái':   ['VIETNT05'],
  'Cẩm Phả':   ['SONTT'],
  'Đông Triều': ['HUONGNV'],
  'Hạ Long':    ['TUNGHX', 'DUYNT', 'TUYENLM', 'CHINH05']
};
```

Infographic tính toán theo địa bàn (không phải GĐV):
- Tổng quan khu vực (HS cần GQ, tỷ lệ GQ, đang tồn)
- Phân tích nghiệp vụ VCX vs TNDS và % tồn >45 ngày
- Bảng điểm nóng (địa bàn Cấp độ 3, tỷ lệ tồn/PS tháng)
- So sánh hiệu suất 4 địa bàn qua thanh bar

**Công thức tỷ lệ đã giải quyết (tyGQ):**
`tyGQ = Math.round(resolved / tongCanGQ * 1000) / 10` → kết quả là % (vd: 75.6%)
⚠️ Lưu ý: phải nhân 1000 rồi chia 10 (không phải nhân 10 chia 10) để ra đúng đơn vị %.

## Màn hình đăng nhập — 2 lựa chọn nguồn dữ liệu

Thay vì chỉ upload file, màn hình upload có 2 card song song:

| Lựa chọn | Mô tả |
|----------|-------|
| **Google Drive** | Ưu tiên `fetch` bản CSV **Xuất bản lên web** (`/pub?gid=…&single=true&output=csv`); lỗi/chưa xuất bản → dự phòng JSONP (`gviz/tq`, callback `ptiGvizCb`) kèm alert cảnh báo |
| **Upload file** | Kéo thả hoặc chọn file `.xlsx` từ máy tính |

- Google Drive file ID: `1KqAQmNh9W-C8MsbWuYrywnNouGC4t2ISwRMa7oYH4_0` · Sheet GID: `101924388` (cố định, không có ô nhập URL)
- File phải chia sẻ "Bất kỳ ai có liên kết" mới tải được
- Hàm tải: `loadFromGoogleDrive()` — luôn dùng `GDRIVE_FILE_ID`, không đọc input từ người dùng. Luồng: fetch CSV xuất bản → `XLSX.read(text,{type:'string',raw:true})` → `_applyDriveRaw(cols)`; nếu fetch/parse lỗi → `_loadDriveViaGviz(fileId)` với cờ `window.__driveViaGvizFallback=true` → `_processGvizData` → `_applyDriveRaw`. Lỗi xử lý chung đi qua `_driveFail(msg)`
- ⚠️ **Vì sao không chỉ dùng gviz (09/2026)**: gviz tự đoán kiểu mỗi cột theo các hàng đầu; cột bị đoán là `number` sẽ trả `null` cho mọi ô dạng chữ (Sheet locale VN giữ `"2,619,000"` là chữ). Thực tế mất 1497/1914 ô `Tiền BT đã trả cho GR` + ô tiền ở `Tiền giảm trừ BT`, `Tiền chia sẻ rủi ro`, `Tiền cứu hộ`, `Tiền TƯ cho KH`, `Tiền BT đã trả cho KH` → KPI Đã chi trả hiện 39.340đ thay vì 15,2 tỷ. Các tham số `range`/`select` không ép được kiểu ổn định.
- ⚠️ `raw:true` khi parse CSV là **bắt buộc** — không có nó SheetJS đổi `"11/09/2026"` thành ngày kiểu Mỹ (tháng/ngày đảo)
- Không dùng `/export?format=csv`: dữ liệu đúng nhưng bước redirect 307 thiếu header CORS → trình duyệt chặn (đã thử Chrome headless từ file://)
- Sheet đã bật **Tệp → Chia sẻ → Xuất bản lên web** (09/2026: sheet `XCG05_HSBT`, CSV, tự động xuất bản lại khi thay đổi). ID xuất bản lưu ở hằng `GDRIVE_PUB_ID` (`2PACX-…`), URL `https://docs.google.com/spreadsheets/d/e/${GDRIVE_PUB_ID}/pub?gid=${GDRIVE_GID}&single=true&output=csv`. Dạng `/d/${GDRIVE_FILE_ID}/pub?...` **không dùng được** (401). Nếu người dùng dừng xuất bản rồi xuất bản lại, ID `2PACX-` có thể đổi → phải cập nhật hằng này. ✅ Từ 23/11/2026 bản xuất bản gồm **toàn bộ tài liệu**, nên `pub?gid=1569610723` (sheet `DoanhthuKenh`) cũng tải được — khối Đợt 3 chạy đường CSV, không còn rơi về gviz. Bản xuất bản có thể trễ vài phút so với Sheet.
- ⚠️ CSV xuất bản **chỉ tải được khi trang chạy qua http/https** (GitHub Pages `https://chinhnhpti013.github.io/baocaotuan/`): bước redirect 307 chỉ trả `Access-Control-Allow-Origin` cho Origin thật, không cho `Origin: null` của file:// → mở `index.html` trực tiếp trên máy sẽ rơi về gviz (có alert giải thích). Đã kiểm tra bằng Chrome headless cả 2 trường hợp.
- ⚠️ URL gviz **bắt buộc** có `&headers=1` để gviz đọc hàng đầu làm tên cột (thiếu tham số này → cột trả về dạng A, B, C, toàn bộ số liệu = 0)
- ⚠️ URL **bắt buộc** có `&_=${Date.now()}` để tránh browser cache: khi cập nhật file Drive mới, không có cache-buster sẽ vẫn tải dữ liệu cũ
- **Không thêm lại ô nhập URL/ID tùy chỉnh** — người dùng cập nhật dữ liệu bằng cách ghi đè file Drive giữ nguyên ID

## Badge ngày dữ liệu (`#file-date-badge`)

Hiển thị trên topbar, màu sắc theo độ cũ:
- 🟢 `fresh`: ≤3 ngày (upload file) / ≤14 ngày (Google Drive)
- 🟡 `warn`: 4–7 ngày (upload) / 4–14 ngày (Drive)
- 🔴 `old`: >7 ngày (upload) / >14 ngày (Drive)

**Upload file**: đọc `file.lastModified` → lưu vào `localStorage` key `pti_filedate`
**Google Drive**: quét `Ngày mở HSBT` tìm max → dùng làm mốc "Dữ liệu đến ngày..."
Hàm: `showFileDateBadge(dateMs, isGDrive)` — gọi sau `renderAll()`

## Lưu phiên làm việc

Dùng `localStorage` với 3 key:
- `pti_raw_v1` — JSON dữ liệu
- `pti_filename` — tên file
- `pti_filedate` — timestamp `lastModified` (chỉ có khi upload file, không có khi dùng Drive)

Trợ lý GĐV dùng thêm 2 key riêng (không bị "↩ Tải file khác" xoá): `pti_ai_log` (hội thoại, tối đa 40 tin, HTML đã render) và `pti_ai_gdv` (mã "GĐV của tôi").

- Khi tải file → tự động lưu cả 3 key
- Khi mở lại trang → nút "🔄 Khôi phục" xuất hiện, tự restore + hiện lại badge
- "↩ Tải file khác" → xoá cache, quay về upload

## Bộ đếm lượt truy cập & người đang xem (`#visit-counter`, 11/2026)

Khối chữ mờ ở **góc dưới bên trái mọi màn hình** (đăng nhập · chọn nguồn · dashboard):
`● Lượt truy cập: 1.248 · Đang xem: 3`. Hướng dẫn thiết lập đầy đủ ở **`docs/bo-dem-truy-cap.md`**.

| Thành phần | Vai trò |
|---|---|
| `VC_DB_URL` | databaseURL của Firebase Realtime Database — đã dán 23/11/2026: project `pti-qn-dashboard`, vùng **asia-southeast1 (Singapore)**. **Để trống → chế độ cục bộ**: chỉ đếm số lần mở trang trên chính máy đó và hiện hậu tố *(máy này)* — cố ý, để trang không bao giờ hiện số sai khi chưa nối được |
| `VC_BEAT` 45s · `VC_TTL` 150s | nhịp báo "còn đang xem" · quá hạn này không có nhịp thì không tính nữa |
| `vcInit()` | nạp 2 file SDK compat từ `gstatic.com` **theo yêu cầu** (chỉ khi có `VC_DB_URL`), lỗi mạng → rơi về `vcLocalMode()` |
| `vcStart()` | `stats/views` +1 bằng `transaction`; `online/<phiên>` ghi mốc giờ máy chủ, `onDisconnect().remove()` xoá khi đóng tab |
| `vcPaint()` · `vcTheme(id)` | vẽ lại 2 con số · đổi tông màu, gọi trong `show(id)` |

- **Vì sao Firebase mà không phải API đếm thường**: "đang xem" cần biết lúc nào một máy **rời** trang — chỉ kết nối thường trực (`onDisconnect`) làm chính xác được. Dịch vụ đếm lượt thuần tuý chỉ ước lượng theo số lượt gần đây.
- **Đặt `position:fixed` ngoài `#dashboard`**, không đặt trong `<footer>`: footer chỉ có ở dashboard, trong khi bộ đếm phải hiện ở cả màn đăng nhập và màn chọn nguồn. `z-index:95` (dưới topbar 100 và popup Trợ lý 1000).
- **Hai tông màu**: màn đăng nhập/upload nền xanh đậm → `rgba(255,255,255,.3)`; dashboard nền sáng → class `.vc-light` `rgba(100,116,139,.42)`. Hover thì rõ hẳn. Người dùng yêu cầu "mờ gần màu nền chân trang nhưng mắt thường vẫn xem được" — **đừng tăng độ đậm** khi dọn CSS.
- Mất mạng → cả 2 số về `—`, **không** giữ số cũ (`.info/connected`). Giờ máy người dùng lệch được bù bằng `.info/serverTimeOffset`.
- ⚠️ **Kiểm thử headless**: `getComputedStyle` đọc ngay sau khi đổi class sẽ ra **màu cũ** vì `transition:color .2s` không được `--virtual-time-budget` tua — phải chèn `*{transition:none!important}` trước khi đo (cùng cái bẫy đã ghi ở mục menu tiêu chí).
- Đo thật 23/11/2026 (Chrome headless, iframe): khối nằm đúng `left 14px / cách đáy 10px` (≤600px: 8px/6px), `overflowX = 0` ở 1400px và 390px, 0 lỗi JS, đổi tông đúng khi `show('dashboard')` ↔ `show('upload-screen')`.
- ⚠️ **Quy tắc bảo mật — kẽ hở đã vá, đừng viết lại bản cũ**: bản đầu dùng `!data.exists() || newData.val() == data.val() + 1`, nghĩa là khi `stats/views` còn **rỗng** thì ghi được **số bất kỳ** (đã dẫm phải thật: một lệnh thử đặt nhầm thành 9999 và chính mình cũng không xoá được, phải vào Console xoá nhánh `stats`). Bản đang dùng bắt buộc **rỗng → chỉ nhận đúng 1**, có rồi → chỉ cộng 1; nhánh `online` thêm `newData.child('t').val() <= now + 60000` để không ai cắm được phiên "sống mãi". Toàn văn ở `docs/bo-dem-truy-cap.md`.
- ⚠️ **Đổi quy tắc xong phải chờ vài phút** mới có hiệu lực (Google ghi rõ "several minutes to fully propagate") — kiểm ngay sau khi Publish sẽ thấy quy tắc **cũ** còn chạy và tưởng là publish hỏng. Kiểm nhanh không cần trình duyệt: `curl -s "<databaseURL>/stats/views.json"` và `curl -s -X PUT --data '<số>' "<databaseURL>/stats/views.json"`.
- Bộ đếm **chỉ gửi đi** 1 nhịp "đang mở trang" và lệnh cộng 1 — không gửi dữ liệu hồ sơ hay thông tin người dùng. Giữ nguyên tính chất này khi sửa.

## Thiết kế UI hiện tại (quan trọng — không thay đổi tuỳ tiện)

### Màu sắc & nền

| Thành phần | Giá trị |
|------------|---------|
| Nền trang | `linear-gradient(150deg, #dce8f5 → #ede7f8 → #f8e9e0 → #fafaf0)`, `background-attachment: fixed` |
| Chart card | `#FAFAF5` (trắng kem), `box-shadow: 0 4px 24px rgba(0,0,0,.08)` |
| Info card (Infographic) | `#fff`, `box-shadow: 0 4px 20px rgba(0,0,0,.06)` |
| CSS variable `--claude` | `#DA7756` (Claude Orange — dùng làm accent) |
| CSS variable `--claude-light` | `#FDF1EE` |

### Tab — 3D Press Button

Mỗi tab có màu gradient riêng, hiệu ứng nổi/nhấn kiểu nút 3D vật lý:

| # | Tab | Màu gradient |
|---|-----|-------------|
| 1 | 📊 Tổng quan | `#60a5fa → #1d4ed8` (xanh dương) |
| 2 | 👤 Theo GĐV | `#34d399 → #047857` (xanh lá) |
| 3 | ⏱️ Tiến độ & Hồ sơ tồn | `#94a3b8 → #334155` (xám thép) |
| 4 | 🛡️ Kiểm soát rủi ro | `#f87171 → #7f1d1d` (đỏ đậm) |
| 5 | 💡 Đề xuất cải tiến CLDV | `#fb7185 → #be123c` (hồng đỏ) |
| — | 🔍 Tra cứu tiến trình | `#fbbf24 → #b45309` (vàng cam) — không có nút tab, màu ở `.dash-dd [data-d="search"]` |
| — | 🧰 Tiện ích | `#818cf8 → #3730a3` (indigo) — không có nút tab, màu ở `.dash-dd [data-d="tienich"]` |
| — | 🧰 Tiện ích | `#818cf8 → #3730a3` (indigo) — không có nút tab, màu này nay nằm ở `.dash-dd [data-d="tienich"]` (mục cuối droplist) |

**Nền nội dung tab (`.tab-pane`)**: mỗi tab có nền gradient pastel dịu cùng tông màu nút tab (xanh dương / xanh lá / tím / cyan / hồng nhạt), bo góc 16px, viền trắng mờ.

**Thanh tab nằm TRONG topbar (11/2026)**: `.tabs#main-tabs` đã chuyển từ `.content` vào trong `.topbar` (topbar vốn `position:sticky;top:0` nên tab luôn hiện khi cuộn — **đã gỡ** rule sticky riêng `top:64px` của `.tabs`).
- Thứ tự trong topbar: logo → `.topbar-center` (tiêu đề + badge ngày) → `.tabs` (đẩy sang phải bằng `margin-left:auto`).
- **Tự co cho vừa 1 hàng**: `fitTopTabs()` thêm dồn `tt-c1…tt-c6` (giảm gap/padding → cỡ chữ `.82rem → .53rem`) tới khi `scrollWidth <= clientWidth` (so **không có dung sai** — khít đúng mép vẫn coi là tràn vì sẽ cắt mất bóng nút 3D); hết mức mà vẫn tràn thì bật `tt-scroll` (cuộn ngang, không cắt mất tab). Gọi ở cuối `renderAll()`, khi `resize` (debounce 120ms) và **sau `document.fonts.ready`** (lần đo đầu webfont chưa tải xong → chọn mức co quá nhẹ). Thoát sớm khi `offsetWidth===0` (đang ở màn đăng nhập/upload) hoặc `innerWidth<=600`.
- **601–1150px**: topbar `flex-wrap:wrap`, thanh tab xuống hàng riêng chiếm trọn bề ngang (ép 1 hàng ở dải này thì hết cả 6 mức co vẫn tràn — đo thật ở 1002px). Đo thật mức co (6 nút tab + tiêu đề "Báo cáo tuần - KV 013" cỡ 1.32rem, số là `window.innerWidth`): 1898/1578px → không co · 1478px → `tt-c1` · 1418/1344px → `tt-c2` · 1258px → `tt-c4` · 1178px → `tt-c5` · ≤1150px → 2 hàng (1138px không co, 1002px `tt-c1`, 778px `tt-c4`). Mọi bề rộng đều `overflowX = 0` và khoảng hở tiêu đề↔tab ≥ 12px, không phải dùng `tt-scroll`.
- ⚠️ **Chụp màn hình bằng Chrome headless đo sai bề ngang**: `--screenshot` phóng viewport ra hết chiều cao trang nên thanh cuộn dọc biến mất → `innerWidth` tăng ~22px, đủ để nhảy qua mốc 1150px và chụp ra cảnh "tab đè lên tiêu đề" không có thật. Muốn kiểm tra thì đọc số đo bằng `--dump-dom` (getBoundingClientRect), hoặc chụp ở bề ngang nằm giữa dải (vd 1100px / 1500px), đừng chụp sát mốc.
- **≤600px**: tab chiếm cả hàng trong topbar, cuộn ngang, `fitTopTabs()` không can thiệp.
- Lưu ý: không thêm `overflow` (hidden/auto/scroll) cho `body` hay `#dashboard` — sẽ làm hỏng `position:sticky` của topbar.

**Khoảng cách dọc (09/2026)**: đã thu hẹp để nhường không gian cho vùng dữ liệu — `.content` padding-top `24px → 10px` (tablet `8px`, mobile `6px`), `.tabs` margin-bottom `24px → 12px` (mobile `8px`).

**Thanh lọc chung đã bị xoá (09/2026)** — 3 bộ lọc GĐV / Nghiệp vụ / Trạng thái cùng `getFiltered()`, `buildGdvFilter()` và toàn bộ CSS `.filter-bar` đã gỡ bỏ. `renderAll()` nay tính thẳng `computeStats(RAW)`; muốn lọc/cắt lát dữ liệu thì dùng khối **Phân tích đa chiều** (7 tiêu chí nhóm, cuối tab Tổng quan) hoặc **Tra cứu tiến trình** (menu thả xuống).

**Topbar rút gọn (11/2026)** — theo yêu cầu người dùng:
- Tiêu đề: **"Báo cáo tuần - KV 013"** (11/2026, rút gọn từ "Báo cáo tuần - Phòng CH& Giám định KV Quảng Ninh"), đặt **cách logo đúng 12mm** (`.topbar-center{margin-left:12mm}`; ≤1150px còn 8mm, mobile 0). `.topbar` đổi `justify-content` thành `flex-start`.
- Cỡ chữ tiêu đề **1.32rem / 700** (tablet 601–900px `1.1rem`, mobile ≤600px `.98rem`); tiêu đề và dòng badge ngày dữ liệu **căn giữa theo nhau** (`.topbar-center{text-align:center}` + `.topbar-meta{justify-content:center}`) — khối này tự co theo nội dung nên "căn giữa" là giữa của chính cặp 2 dòng, không phải giữa topbar.
- ⚠️ `.topbar-center{flex:0 0 auto}` + `padding-right:16px`: tiêu đề `white-space:nowrap` mà cho co (`flex-shrink:1`) thì chữ **tràn đè lên thanh tab** (flex item vẫn vẽ phần vượt khung). Thiếu chỗ thì để `fitTopTabs()` co thanh tab, đừng co khối tiêu đề.
- **Đã gỡ khỏi topbar**: pill số hồ sơ `#record-count` (và dòng gán nó trong `renderAll()`), nút "↩ Tải file khác" (`.btn-reset`), CSS `.topbar-right`/`.topbar-count`. **23/11/2026 dọn nốt mã chết**: hàm `resetApp()` và CSS `.btn-reset` đã xoá hẳn (không còn nút nào gọi tới). Muốn nạp lại dữ liệu thì dùng nút **🔄 Làm mới** (`#btn-refresh`); muốn đổi hẳn nguồn thì xoá cache trình duyệt.
- Tên nguồn dữ liệu ("Google Drive — Dulieu.xlsx" / tên file upload) **không hiển thị**: `#topbar-file` vẫn còn trong DOM nhưng `display:none` để `_applyDriveRaw()`/`handleFile()` ghi vào không lỗi. Badge ngày dữ liệu `#file-date-badge` giữ nguyên.

- Trạng thái mặc định: `transform: translateY(-3px)`, `box-shadow: 0 6px 0 rgba(0,0,0,0.22)`
- Hover: `translateY(-5px)`, shadow `8px`
- Active/nhấn: `translateY(+2px)`, shadow `2px`

### Biểu đồ — Chart.js 4

**Bar chart (cột trụ 3D — sắc nét):**
- `cylinderPlugin`: nắp đỉnh = ellipse màu **đặc** sáng hơn màu cột (`shadeHex` +0.55→+0.12, gradient ngang) + viền đậm sắc nét (`shadeHex` −0.28, lineWidth 1.2); đáy = nửa ellipse dưới màu đậm của cột. **Không dùng radial gradient trắng mờ.**
- Màu nắp lấy từ `ds._cyl` (`{top,bot}` hex) — `buildBar` tự gắn từ `GRAD_PALETTE`; nếu `backgroundColor` là mảng hex per-bar thì nắp theo từng bar
- Gradient thân cột: đậm ở đáy → màu gốc (55%) → `shadeHex(top, 0.3)` sáng đặc ở đỉnh (không còn stop trắng trong suốt)
- `GRAD_PALETTE` 7 màu (indigo/emerald/amber/red/violet/cyan/orange)
- `borderRadius: {topLeft:0, topRight:0, bottomLeft:5, bottomRight:5}`
- Animation: `easeOutQuart`, 1000ms

**Chart TLBT XO (`_buildDualNvBtChart`)**: dual-axis bar (doanh thu trái, TLBT% phải) — đồng dạng `buildBar`: gradient cột đậm→sáng, `cylinderPlugin` (datasets gắn `_cyl` thủ công), `borderRadius` đáy 5px, animation `easeOutQuart` 1000ms. TLBT (%) hiển thị **2 số thập phân** (`toFixed(2)`) ở cả bảng và tooltip chart; `parsePct` làm tròn 2 chữ số.

**Donut chart (vòng tròn 3D — ánh kim):**
- Gradient kim loại theo bán kính từng múi (`shadeHex`): sáng ở trong → màu gốc → tối ở mép
- `metallicSheenPlugin`: dải sáng chéo trắng + viền sáng mép ngoài trên vành donut
- CSS transform trực tiếp trên canvas: `perspective(480px) rotateX(24deg) scaleY(0.86)`
- Hover chuột: `rotateX(12deg)` — ngẩng lên
- `shadow3DPlugin`: đổ bóng canvas
- Animation: `easeOutBack`, 1100ms, `animateScale: true`

### Quy tắc khi Claude chỉnh sửa index.html

- Không thay đổi cấu trúc upload → loading → dashboard
- Không thay đổi hàm `computeStats()` trừ khi thêm trường mới — hàm trả về `rows: data` (mảng đã lọc) để các hàm render dùng thay vì `RAW`
- Khi thêm tab mới: thêm cả button, div container, case switchTab, hàm render, và gọi trong renderAll()
- Tiêu đề topbar: **Báo cáo tuần - KV 013** (footer vẫn giữ **PTISOS - Văn phòng Miền Bắc - Phòng Khu vực Quảng Ninh**)
- Màu brand PTI: `#1565C0` · Font: `Be Vietnam Pro`
- **Không** thay đổi `cylinderPlugin`, `shadow3DPlugin`, `GRAD_PALETTE` trừ khi người dùng yêu cầu
- Khi thêm chart bar mới → dùng `buildBar()`, chart donut mới → dùng `buildDonut()`

## Quy ước ngưỡng ngày tồn (QUAN TRỌNG)

| Bucket | Điều kiện | Ghi chú |
|--------|-----------|---------|
| 1–30 ngày | `n >= 1 && n <= 30` | |
| 31–45 ngày | `n > 30 && n <= 45` | |
| 46–89 ngày | `n > 45 && n < 90` | |
| ≥ 90 ngày | `n >= 90` | Ngưỡng cảnh báo — dùng `>=90` KHÔNG phải `>90` |

- `qua90`: `parseTon(...) >= 90` (dùng nhất quán toàn bộ file)
- `g45_90`: `n > 45 && n < 90` (không bao gồm 90)
- `g90 / b4`: `n >= 90` (bao gồm đúng 90 ngày)
- Hàm `gdvKey(r)` phải dùng khi lấy mã GĐV thay vì `r['GĐV thụ lý']` trực tiếp

## Hàm helper dùng chung

- `parseDate(v)` — hàm global duy nhất parse ngày (Excel serial, Date, string). Không tạo bản sao cục bộ trong các hàm khác.
- `gdvKey(r)` — lấy mã GĐV từ `r['GĐV thụ lý'] || r['Mã GĐV']`
- `parseTon(v)` — parse số ngày tồn, trả về `null` nếu `n <= 0`
- `parseMoney(v)` — parse số tiền, trả về `0` nếu không hợp lệ

## Đặc tả 3 báo cáo gốc BC1 / BC2 / BC3 (nay đã gộp)

> 11/2026: tab Báo cáo XO đã bỏ. **BC1 gộp với BC4 → `garageSection()`**, **BC2 + BC3 gộp với Top 50 đứng yên → `tonListSection()`**, cả hai nằm ở tab ⏱️ Tiến độ & Hồ sơ tồn. Phần dưới giữ lại vì là **đặc tả gốc của truy vấn và bộ cột** — khi sửa 2 hàm gộp phải bảo đảm vẫn đủ các cột và điều kiện lọc dưới đây.

### BC1 — Thống kê Doanh thu SC các Gara/Showroom (PS năm hiện tại)

Tham chiếu Query: `select [Tên garage (thường gọi)], Count([Tiền ước/duyệt BT]), sum([Tiền ước/duyệt BT]), sum([Tiền BT đã trả cho GR]) Where [Ngày mở HSBT] >= date 'năm-01-01' group by [Tên garage (thường gọi)] order by sum([Tiền ước/duyệt BT]) DESC limit 10`

> ⚠️ Vị trí cột Excel (A/B/C...) không cố định giữa các lần xuất file — mọi truy xuất dữ liệu trong code PHẢI dùng tên cột (header), không dùng chỉ số/chữ cái cột. Bảng dưới chỉ để tham chiếu tên cột, không phải vị trí.

| Cột | Nguồn | Logic |
|-----|-------|-------|
| Tên Gara/SH | `Tên garage (thường gọi)` (fallback `Tên garage (ĐKKD)`) | Group by |
| Số vụ phát sinh | `Tiền ước/duyệt BT` | Count |
| Tiền SC ước BT | `Tiền ước/duyệt BT` | Sum |
| Tiền SC đã BT | `Tiền BT đã trả cho GR` | Sum |

Điều kiện lọc: `Ngày mở HSBT` ≥ 01/01/năm hiện tại · Sắp xếp giảm dần theo tổng tiền ước BT · **Top 30** · Khung cố định `max-height:480px` + scroll dọc, thead sticky

### BC2 — Danh sách chi tiết HS tồn ≥ 90 ngày

Tham chiếu Query: `Select [GĐV thụ lý], [Số HSBT], [Biển số xe], [Mã nghiệp vụ], [Tên garage (thường gọi)], [Mã check], [Mã validate], [Tiền ước/duyệt BT], [Trạng thái hồ sơ], [Số ngày tồn] WHERE [Số ngày tồn]>=90 Order by [Số ngày tồn] DESC`

| Tên cột | Ghi chú |
|---------|---------|
| GĐV thụ lý | dùng qua `gdvKey(r)` |
| Số HSBT | |
| Biển số xe | |
| Mã nghiệp vụ | |
| Tên garage (thường gọi) | fallback `Tên garage (ĐKKD)` |
| Mã check | |
| Mã validate | |
| Tiền ước/duyệt BT | |
| Trạng thái hồ sơ | |
| Số ngày tồn | Điều kiện ≥ 90 |

Tô màu: 🔴 ≥ 180 ngày · 🟡 120–179 ngày · Sắp xếp ngày tồn giảm dần · Header đỏ

### BC3 — Danh sách chi tiết HS tồn >45 và ≤90 ngày

Cùng cấu trúc cột với BC2, lọc: `n > 45 && n < 90` · Header màu vàng/cam · Khung `max-height:480px` + scroll dọc

Helper dùng chung: `makeHsTonRows(list)` và `hsTonHeader` string — tái sử dụng cho cả BC2 và BC3
