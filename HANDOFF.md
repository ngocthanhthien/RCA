# HANDOFF — RCA Tracking App

Tài liệu bàn giao cho **RCA Tracking App** — ứng dụng quản lý RCA (Root Cause Analysis) & Kế hoạch hành động khắc phục cho ILD Coffee Vietnam.

- **File chính**: `index.html` (GitHub Pages: https://ngocthanhthien.github.io/RCA/) — **1 file HTML duy nhất**, tự chứa toàn bộ HTML/CSS/JS + dữ liệu gốc, chạy 100% offline (mở trực tiếp bằng trình duyệt, không cần server/cài đặt).
- **Kích thước hiện tại**: ~3.400 dòng, ~300KB.
- **Không dùng framework, không dùng thư viện ngoài, không build step.** Vanilla HTML/CSS/JS thuần, mọi thứ nằm trong 1 file để dễ copy/chia sẻ/mở lại nhiều năm sau mà không sợ mất phụ thuộc.

---

## 1. Cách chạy / mở app

Double-click `index.html` → mở bằng **Google Chrome hoặc Microsoft Edge** (bắt buộc cho các tính năng Evidence/Đồng bộ thư mục — xem mục 6). Firefox/Safari vẫn dùng được phần lớn tính năng, chỉ thiếu 2 tính năng đó.

Không cần cài gì thêm, không cần internet (trừ khi dùng font hệ thống lạ — không áp dụng ở đây).

---

## 2. Nguồn dữ liệu gốc

Dữ liệu khởi tạo lần đầu (biến `SEED_DATA` nhúng thẳng trong file, ~64 RCA + ~145 hành động) được import từ:

- `RCA TRACKING FILE 2026.xlsx` — sheet **"Master file"** (danh sách RCA) và **"Action plan"** (kế hoạch hành động gốc).
- `RCA Template (new-25.12.2025).xlsx` — file mẫu in ấn "BÁO CÁO PHÂN TÍCH NGUYÊN NHÂN GỐC RỄ", dùng để dựng chức năng xuất PDF/Excel theo đúng biểu mẫu công ty (mục 8). Logo "iLD" trong báo cáo được trích xuất trực tiếp từ file này.

Sau lần nạp đầu tiên, **mọi thay đổi chỉ tồn tại trong `localStorage` của trình duyệt** — sửa lại 2 file Excel trên **không** ảnh hưởng gì tới app nữa (trừ khi bấm "Khôi phục dữ liệu gốc từ Excel" ở tab Cài đặt, lúc đó `SEED_DATA` nhúng sẵn trong HTML được nạp lại).

---

## 3. Kiến trúc & luồng dữ liệu

```
localStorage["rca_tracking_app_v1"]   <-- nguồn sự thật chính (rcaList, actionList, settings)
        |
        | persist() ghi mỗi khi có thay đổi
        v
   state (biến JS toàn cục, in-memory)
        |
        | render*() đọc state, vẽ lại DOM
        v
   UI (4-6 tab, tuỳ quyền đăng nhập)
```

- **Không có backend, không có API.** Toàn bộ logic chạy client-side trong trình duyệt.
- **`state`** là 1 object JS toàn cục giữ: `rcaList[]`, `actionList[]`, `settings{}`, `rcaFilters{}`, `actFilters{}`, `rcaSort{}`, `actSort{}`.
- Mọi hàm `render*()` (renderDashboard, renderRcaTable, renderActionTable, renderSettingsTab...) đọc `state` và vẽ lại DOM tương ứng — **không dùng framework reactive nào**, gọi `render*()` thủ công sau mỗi lần đổi `state`.
- `persist()` là điểm ghi trung tâm duy nhất xuống `localStorage`, đồng thời kích hoạt đồng bộ ra thư mục (nếu đã cấu hình — mục 6).

### Bản đồ code (theo section comment trong file)

| Dòng ~ | Section | Nội dung |
|---|---|---|
| 1078 | SEED DATA | Dữ liệu gốc import từ Excel, nhúng dạng JSON literal |
| 1081 | CONFIG | Hằng số: danh mục mặc định, `localStorage` keys, mật khẩu Admin mặc định |
| 1106 | STATE | Định nghĩa `state`, `loadState()`, `persist()` |
| 1162 | AUTH | Đăng nhập/đăng xuất, phân quyền (`isAdmin`, `canEditRca`...) |
| 1310 | UTIL | Hàm tiện ích chung (format ngày, escape HTML, toast...) |
| 1381 | EVIDENCE | Đính kèm file theo RCA qua File System Access API |
| 1654 | AUTO-SYNC | Đồng bộ dữ liệu tự động ra thư mục local (File System Access API) |
| 1796 | PRESENTATION MODE | Nút A-/A+/Đậm — phóng to cỡ chữ toàn app |
| 1838 | SETTINGS | Logic tab Cài đặt (Admin-only) |
| 1936 | DROPDOWN POPULATION | Đổ dữ liệu vào các dropdown/multi-select filter |
| 2087 | DASHBOARD | Tab Tổng quan: KPI, biểu đồ canvas |
| 2195 | EMAIL REPORT | Sinh file `.eml` kèm biểu đồ + bảng dữ liệu |
| 2525 | RCA LIST | Lọc/sort/render bảng Danh sách RCA |
| 2641 | ACTION PLAN LIST | Lọc/sort/render bảng Kế hoạch hành động |
| 2721 | VIEW RCA DETAIL | Popup xem chi tiết 1 RCA |
| 2782 | RCA TEMPLATE REPORT | Xuất PDF (in) + Excel theo đúng biểu mẫu công ty |
| 2987 | MODALS: RCA | Form thêm/sửa RCA (bao gồm checklist hành động) |
| 3184 | MODALS: ACTION | Form thêm/sửa 1 hành động |
| 3291 | FILTER WIRING | Gắn sự kiện cho ô tìm kiếm |
| 3302 | IMPORT / EXPORT | Xuất/nhập JSON, xuất CSV |
| 3400 | INIT | `init()` — điểm khởi động app khi tải trang |

---

## 4. Data model

### RCA record (`state.rcaList[]`)

```
id, stt, rcaNumber, status, function, dept, leader, classification,
issueDate, completeDate, category, defectMode, issue, problemStatement,
rootCause, action, actionItems[], standards, fourM,
result1, result2, result3, evidence[]
```

- **`action`** (text) là bản tóm tắt **tự sinh** từ `actionItems[]` khi lưu — không nhập tay trực tiếp nữa.
- **`actionItems[]`** = `{id, content, pic, dueDate}[]` — checklist hành động nhập ngay trong form RCA. **`id` của mỗi dòng chính là `id` thật của bản ghi tương ứng trong `state.actionList`** (xem mục 5) — đây là cơ chế liên kết 2 chiều, không dùng khoá ngoại riêng.
- **`evidence[]`** = `{name, size, type, addedAt, addedBy}[]` — chỉ lưu **metadata**, file thật nằm trên ổ đĩa (thư mục Evidence, mục 6), không nhúng vào `localStorage`.
- `status` lưu 1 trong 4 giá trị tĩnh: `On Progress | Late | Done | Cancel`. **`Late` cũng có thể tự động suy ra** (xem `effectiveRcaStatus()`) khi RCA còn `On Progress` nhưng đã quá hạn theo `classification` + `settings.classDays`.

### Action record (`state.actionList[]`)

```
id, rcaNumber, issue, correctiveAction, pic, status, function, dept,
dueDate, completeDate, evidence, verifyEffective, remarks
```

- Liên kết với RCA qua **`rcaNumber`** (chuỗi, không phải id) — đây là khoá nối chính giữa 2 tab Danh sách RCA và Kế hoạch hành động.
- Một action được xem là "thuộc checklist của RCA" nếu `id` của nó xuất hiện trong `rcaList[x].actionItems[].id` nào đó (hàm `isActionLinkedToRca()`). Nếu có, 3 trường **content/PIC/dueDate bị khoá** trong form sửa action — phải sửa từ form RCA (tab Danh sách RCA), tránh 2 nơi ghi đè nhau. Trường **status/completeDate/evidence/verifyEffective/remarks** luôn sửa được ở tab Kế hoạch hành động.
- Action **không** thuộc checklist nào (thêm tay qua nút "+ Thêm Action") sửa tự do hoàn toàn.
- **Status action cập nhật 2 chiều**: ngoài tab Kế hoạch hành động, còn đổi được ngay tại (1) popup xem chi tiết RCA — dropdown ở mục "Corrective actions", lưu ngay (người có `canEditAction()`), và (2) cột Status trong checklist của form sửa RCA — lưu khi bấm Lưu RCA. Status **không** lưu trong `actionItems[]`; nguồn sự thật duy nhất vẫn là `state.actionList[].status` (`withLiveStatus()` đọc vào form, `applyActionStatus()` ghi ra). Chuyển sang `Done` mà chưa có Complete Date thì tự điền ngày hôm nay.

### `state.settings` (Admin-editable, tab Cài đặt)

```
functions[], depts[], categories[], classDays{Minor,Major,Serious}, leaders[], companyName
```

Mặc định lấy từ `DEFAULT_FUNCTIONS/DEFAULT_DEPTS/DEFAULT_CATEGORIES/DEFAULT_CLASS_DAYS/DEFAULT_COMPANY_NAME` (khai báo ở đầu file). `settings.leaders` là danh sách Leader **thêm tay** — cộng dồn (union) với danh sách Leader tự suy ra từ `rcaList[].leader` để tạo dropdown đăng nhập.

---

## 5. Phân quyền (Auth)

3 mức, không có backend xác thực thật — toàn bộ kiểm tra quyền chạy **client-side** (đủ cho nội bộ, **không phải bảo mật cấp doanh nghiệp** — ai mở DevTools cũng sửa được `state` trực tiếp).

| Vai trò | Đăng nhập | Quyền |
|---|---|---|
| **Admin** | Nhập mật khẩu (mặc định `QAILD`, đổi được ở tab Cài đặt → lưu riêng key `rca_tracking_app_v1_admin_password`, **không** nằm trong file backup JSON) | Toàn quyền + tab Cài đặt |
| **Leader** | Chọn tên từ dropdown (không mật khẩu) | Thêm RCA mới; sửa RCA/action mà mình đứng tên Leader **hoặc** có tên trong PIC của 1 action thuộc RCA đó (`isRelatedToRca()`); xem mọi thứ khác |
| **Khách** (chưa đăng nhập) | — | Chỉ xem |

Phiên đăng nhập lưu ở `sessionStorage` (mất khi đóng tab/trình duyệt, không phải `localStorage`).

---

## 5b. Chế độ Firebase (đăng nhập tài khoản + dữ liệu dùng chung) — thêm 28/09/2026

Bật khi hằng `FIREBASE_CONFIG` (section **CLOUD**, ngay sau AUTH) có `apiKey`. Để `null` → app chạy y như mục 5 (localStorage + mật khẩu Admin cục bộ). Mô phỏng theo Sensory-App nhưng **project Firebase riêng**.

- **Đăng nhập 2 cấp**: màn hình khoá ban đầu hỏi **mật khẩu chung** (tài khoản Firebase `viewer@ild-rca.local`, vai trò **Chỉ xem**). Tài khoản riêng `tên@ild-rca.local` (Leader/Admin) đăng nhập ở nút tên góc phải hoặc link "tài khoản riêng" trên màn hình khoá. Chủ dự án `dangthanhbinh53@gmail.com` đăng nhập Google, luôn là Admin.
- **Hồ sơ quyền** `rca_users/{uid}`: `{username, email, displayName, leaderName, role: viewer|leader|admin, active, shared, createdAt, createdBy}`. `leaderName` phải trùng tên trong cột Leader/PIC — `authApply()` gán `currentUser = {role, name: leaderName}` nên toàn bộ `canEditRca/canEditAction…` cũ dùng lại nguyên vẹn. Viewer → `currentUser = null`.
- **Tab 🛡 Người dùng** (Admin): tạo/đổi mật khẩu chung, tạo tài khoản riêng, đổi vai trò, sửa tên Leader, khoá/mở, đổi mật khẩu (cần biết mật khẩu cũ — giới hạn gói Spark), xoá quyền (tài khoản Auth vẫn còn, xoá hẳn ở Console). Tạo/đổi mật khẩu dùng app Firebase phụ in-memory (`authSecondary`) để không đăng xuất Admin.
- **Dữ liệu**: `rca_records/{id}`, `rca_actions/{id}`, `rca_meta/settings` (field `_updatedAt/_updatedBy` là metadata, bị bỏ khi đọc). `persist()` → `cloudSchedulePush()` (debounce 400ms) so JSON chuẩn hoá (`stableJson`) với `CLOUD.known` → chỉ ghi document đổi; Admin mới phát lệnh xoá và ghi settings. `onSnapshot` 3 collection → ghi đè `state` + `persist(true)` (chỉ cache localStorage) + `cloudRenderAll()`. Offline do Firestore `persistentLocalCache` tự xếp hàng ghi.
- **Lần đầu**: Firebase trống → chủ dự án/Admin được hỏi đẩy dữ liệu trên máy lên (hoặc nút "Đẩy dữ liệu lên Firebase (lần đầu)" ở tab Dữ liệu & Cấu hình). Collection rỗng mà chưa từng có dữ liệu thì **không** xoá state trên máy.
- **Quyền thật** ở `firestore.rules` (trong repo): thành viên đọc; Leader/Admin tạo/sửa RCA & Action; chỉ Admin xoá, ghi `rca_meta`, quản lý `rca_users`. Giới hạn: Rules chưa kiểm "Leader chỉ sửa RCA có tên mình" — việc này chỉ do giao diện chặn.
- Xung đột: ghi đè theo từng bản ghi (bản lưu sau thắng) — đủ cho quy mô hiện tại, không có gộp 3 chiều như Sensory.
- Evidence, đồng bộ thư mục, JSON backup vẫn chạy cục bộ như cũ.

### Cài đặt Firebase (1 lần)
1. Firebase Console → tạo project mới → thêm **Web app** → copy `firebaseConfig` dán vào `const FIREBASE_CONFIG = {...}` trong `index.html`.
2. Authentication → Sign-in method: bật **Email/Password** và **Google**. Settings → Authorized domains: thêm `ngocthanhthien.github.io`.
3. Firestore Database → Create database (location châu Á) → tab Rules: dán nội dung `firestore.rules` → Publish.
4. Push `index.html` lên GitHub, mở `https://ngocthanhthien.github.io/RCA/` → "Đăng nhập tài khoản riêng" → Google (chủ dự án) → đồng ý đẩy dữ liệu lần đầu (nếu dữ liệu mới nhất ở file JSON: Huỷ → Phục hồi JSON → bấm nút đẩy ở tab Dữ liệu & Cấu hình).
5. Tab 🛡 Người dùng: tạo mật khẩu chung + tài khoản Leader/Admin.

---

## 6. Tính năng theo tab

- **📊 Tổng quan** — KPI, biểu đồ (donut trạng thái, cột theo Function/Category/tháng), danh sách action sắp/đã quá hạn, nút "Gửi email báo cáo" (tải file `.eml` có biểu đồ PNG canvas nhúng base64 + bảng đầy đủ, không cần hỏi/xem trước).
- **📋 Danh sách RCA** — bảng có: multi-select filter theo 7 trường (Function/Dept/Status/Category/Classification/Leader/Tháng), sort theo mọi cột (mặc định STT tăng dần), tìm kiếm tự do, click cả dòng để xem chi tiết, nút 👁 xem / ✎ sửa / +Act / 📎 evidence riêng từng dòng.
- **✅ Kế hoạch hành động** — bảng hành động, filter + sort tương tự (mặc định sort theo RCA Number).
- **📖 Hướng dẫn sử dụng** — tài liệu hướng dẫn ngay trong app, ưu tiên nội dung cho Leader.
- **🗂 Dữ liệu & Cấu hình** (mọi người dùng được) — Sao lưu/Phục hồi JSON thủ công, Xuất CSV, **Đồng bộ tự động ra thư mục** (chọn 1 lần qua File System Access API, mỗi lần `persist()` tự ghi đè `RCA_Data_Backup.json` vào thư mục đó), **Thư mục Evidence** (mỗi RCA có 1 thư mục con trùng tên mã RCA để lưu file đính kèm thật trên ổ đĩa).
- **⚙️ Cài đặt** (chỉ Admin) — đổi mật khẩu Admin, tên công ty, hạn xử lý theo Classification, quản lý danh mục Function/Dept/Category, thêm Leader mới, nút "Khôi phục dữ liệu gốc từ Excel" (reset toàn bộ về `SEED_DATA`).
- **Xuất báo cáo 1 RCA** (nút trong popup xem chi tiết) — PDF (mở tab mới, có nút in/lưu PDF) và Excel (`.xls` dạng HTML-Excel, nền trắng, có logo), theo đúng bố cục `RCA Template (new-25.12.2025).xlsx`. Mục "5WHY" và bảng "Correction tạm thời" luôn để trống (app chưa có dữ liệu cấu trúc cho 2 mục này — xem mục 9).
- **A-/A+/Đậm** (góc phải header) — phóng to/thu nhỏ + in đậm toàn app, hữu ích khi trình chiếu.

---

## 7. `localStorage` keys

| Key | Nội dung | Có trong file backup JSON? |
|---|---|---|
| `rca_tracking_app_v1` | `{rcaList, actionList, settings}` — dữ liệu chính | ✅ (đây chính là nội dung export) |
| `rca_tracking_app_v1_session` (sessionStorage) | Phiên đăng nhập hiện tại | ❌ |
| `rca_tracking_app_v1_admin_password` | Mật khẩu Admin đã đổi (nếu có) | ❌ — cố ý tách riêng để không lộ khi chia sẻ file backup |
| `rca_tracking_app_v1_uiscale` / `..._uibold` | Cỡ chữ / chế độ đậm | ❌ (theo máy) |
| `rca_tracking_app_v1_email_from/to/cc` | Nhớ email lần gửi báo cáo trước | ❌ |
| IndexedDB `rca_evidence_fs_db` | `FileSystemDirectoryHandle` của thư mục Evidence & thư mục Đồng bộ | ❌ (không thể serialize qua JSON) |

**Ý nghĩa quan trọng**: xuất file JSON backup (nút "Sao lưu") **không** mang theo mật khẩu Admin, cấu hình thư mục Evidence/Đồng bộ, hay cỡ chữ — những thứ này phải cấu hình lại thủ công trên mỗi máy/trình duyệt mới.

---

## 8. Yêu cầu trình duyệt

- **Bắt buộc Chrome/Edge** cho: đính kèm Evidence, Đồng bộ tự động ra thư mục (cả 2 dùng File System Access API — `showDirectoryPicker`). Trên Firefox/Safari, 2 nút này hiện cảnh báo "không hỗ trợ" thay vì lỗi vỡ trang.
- Mọi tính năng khác (CRUD, filter, sort, dashboard, xuất PDF/Excel/CSV/JSON/email) chạy trên **mọi trình duyệt hiện đại**.
- `zoom` CSS (dùng cho nút A-/A+) là thuộc tính không chuẩn nhưng được hầu hết trình duyệt hiện đại hỗ trợ; nếu trình duyệt không hỗ trợ thì chỉ đơn giản là không phóng to được, không lỗi.

---

## 9. Quyết định thiết kế đã chốt (đọc trước khi "sửa lại cho đúng file mẫu")

Khi đối chiếu với `RCA Template (new-25.12.2025).xlsx`, các mục sau **cố ý để trống** vì app chưa thu thập dữ liệu tương ứng — đã thống nhất với người dùng, không phải thiếu sót:

1. **Mục "3. Phân tích 5WHY"** (bảng 5 cột Tại sao + Đúng/Sai theo từng 4M) — app chỉ có 1 trường `rootCause` dạng văn bản tự do, không có cấu trúc 5-Why. Báo cáo chỉ in khung bảng trống.
2. **Bảng "Correction (hành động khắc phục tạm thời)"** — app không phân biệt hành động tạm thời/lâu dài, toàn bộ action đều đổ vào bảng "Corrective & preventive action" bên dưới.
3. **4 ô tích Safety/Quality/Volume/Cost** — chỉ tích khi `category` khớp **chính xác** 1 trong 4 tên này; 4 category khác của app (Output, Environment, Green Bean, WWT, SHE) sẽ không tích ô nào, không có ghi chú thêm.
4. **Mục "Corrective actions"** trong popup xem chi tiết và bảng "Corrective & preventive action" trong báo cáo PDF/Excel **phải luôn khớp nhau 100%** — cả 2 cùng lọc `state.actionList` theo `rcaNumber` (đã có lần bị lệch do 1 bên chỉ đọc `actionItems`, đã sửa — xem hàm `getRcaTemplateData()`).

Nếu sau này muốn bổ sung dữ liệu 5-Why có cấu trúc hoặc phân loại Tạm thời/Lâu dài cho action, đây là 2 việc **mở rộng data model**, không phải sửa lỗi.

---

## 10. Việc bảo trì thường gặp

- **Thêm/xoá Function, Dept, Category, đổi hạn xử lý theo Classification, thêm Leader mới**: tab Cài đặt (Admin) → không cần sửa code.
- **Đổi tên công ty / logo trên header app**: tab Cài đặt → "Thông tin công ty" đổi được tên; **logo trong báo cáo PDF/Excel là ảnh PNG base64 hard-code** trong biến `RCA_LOGO_BASE64` (đầu section "RCA TEMPLATE REPORT") — muốn đổi logo phải thay base64 này (nên dùng script trích xuất + `base64.b64encode` để tránh gõ tay gây lệch chuỗi — đã từng bị lỗi này 1 lần, xem git history / hội thoại).
- **Đổi mật khẩu Admin mặc định** (khi làm mới hoàn toàn / reset máy khác): sửa `let ADMIN_PASSWORD = 'QAILD';` ở đầu file (section CONFIG) — chỉ áp dụng cho máy **chưa từng đổi mật khẩu qua UI** (vì UI ghi đè localStorage riêng).
- **Nạp lại dữ liệu gốc từ Excel** (khi có bản Excel mới hoàn toàn thay vì chỉnh tay từng RCA): phải chạy lại quy trình import ban đầu (đọc Excel bằng `openpyxl` → build JSON → nhúng vào biến `SEED_DATA`) — không có nút tự động trong app, cần thao tác code.

---

## 11. Việc còn để ngỏ / có thể làm tiếp

- Dữ liệu 5-Why có cấu trúc (mục 9.1) — nếu công ty cần đúng biểu mẫu 100%.
- Phân loại "Tạm thời / Lâu dài" cho từng hành động (mục 9.2).
- Test PDF `.xls` "Excel" thực chất là **HTML giả dạng .xls** (kỹ thuật hợp lệ, Excel mở được nhưng có cảnh báo "định dạng không khớp phần mở rộng" — bấm Yes để mở bình thường), không phải file nhị phân `.xlsx` thật.
- Chưa test trên Outlook mới (new Outlook) / Outlook Web / Mail trên Mac cho tính năng gửi file `.eml` — chỉ xác nhận hoạt động tốt trên Outlook desktop (classic).

---

*Tài liệu này mô tả trạng thái app tại thời điểm bàn giao (14/09/2026). Khi đọc lại sau này, ưu tiên đối chiếu với code thật — tài liệu có thể lệch nếu có thay đổi sau đó mà quên cập nhật file này.*
