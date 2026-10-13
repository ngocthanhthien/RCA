# HANDOFF — RCA Tracking App

Ứng dụng quản lý **RCA (Root Cause Analysis) & Kế hoạch hành động khắc phục** cho ILD Coffee Vietnam.

| | |
|---|---|
| File chính | `index.html` — **1 file HTML duy nhất** chứa toàn bộ HTML/CSS/JS + dữ liệu gốc (~4.190 dòng, ~350 KB) |
| Chạy online | GitHub Pages: https://ngocthanhthien.github.io/RCA/ (repo public `ngocthanhthien/RCA`, nhánh `main`, deploy tự động ~30–60 giây sau khi push) |
| Dữ liệu dùng chung | Firebase project **`rca-tracking-ild`** (Firestore `asia-southeast1`, gói Spark miễn phí) |
| Chủ dự án | `dangthanhbinh53@gmail.com` (chủ Firebase project, luôn là Admin trong app) |
| Công nghệ | Vanilla HTML/CSS/JS, **không framework, không build step**. Thư viện ngoài duy nhất: Firebase JS SDK 12.19.0 import động từ `gstatic.com` |
| Cập nhật tài liệu | 06/10/2026 |

---

## 0. Đọc trước khi sửa (dành cho AI tiếp nhận)

1. **Giữ app dạng 1 file `index.html`**, không tách file, không thêm framework/bundler.
2. **Không đọc nguyên file một lần**: dòng ~1170 (`const SEED_DATA = {...}`) là một dòng JSON rất dài, và các biến `RCA_LOGO_BASE64` dài. Dùng tìm kiếm theo section comment `/* ================= TÊN ================= */` rồi đọc theo khoảng dòng.
3. **Không đổi** key `localStorage` (`rca_tracking_app_v1…`), tên collection Firestore (`rca_records`, `rca_actions`, `rca_meta`, `rca_users`), cấu trúc record (mục 4) — dữ liệu thật đang nằm trên Firebase và trong trình duyệt người dùng.
4. Sau khi sửa JS, **kiểm tra cú pháp**:
   `node -e "const h=require('fs').readFileSync('index.html','utf8');new Function(h.match(/<script>([\s\S]*?)<\/script>/)[1]);console.log('OK')"`
5. Người dùng giao tiếp bằng **tiếng Việt**; mọi nhãn giao diện là tiếng Việt, giữ thuật ngữ RCA / PIC / 4M / Corrective action.
6. Người dùng thường **tự commit & push**. Chỉ commit/push khi được yêu cầu rõ.
7. File dùng CRLF trong git (Windows). Khi ghi file bằng script, giữ nguyên nội dung ngoài phần sửa; kiểm tra `git diff --stat` chỉ đổi đúng các dòng cần đổi.

---

## 1. Cách chạy

- **Online (cách dùng chính)**: mở https://ngocthanhthien.github.io/RCA/ → màn hình đăng nhập.
- **Mở file trực tiếp** (`file://`): vẫn chạy, nhưng Firebase có thể bị chặn ngoài tên miền được phép → nên dùng bản online.
- **Chạy thử cục bộ**: `python -m http.server 8767` trong thư mục repo → http://localhost:8767 (`localhost` nằm trong Authorized domains của Firebase).
- Trình duyệt: **Chrome/Edge** (bắt buộc cho Evidence và Đồng bộ thư mục — File System Access API). Firefox/Safari dùng được phần còn lại.
- ⚠ Không xem thử bằng URL `data:` (một số khung preview làm vậy): `localStorage` bị cấm → app khởi động lỗi. Phải phục vụ qua `http://`.

---

## 2. Trạng thái hiện tại (06/10/2026)

| Hạng mục | Trạng thái |
|---|---|
| Firebase project `rca-tracking-ild` | ✅ Đã tạo; Web app "RCA Tracking Web"; Email/Password + Google đã bật; Authorized domain `ngocthanhthien.github.io` đã thêm; Firestore đã tạo ở `asia-southeast1`; Rules đã Publish (nội dung = `firestore.rules`) |
| `FIREBASE_CONFIG` trong `index.html` | ✅ Đã điền → app chạy **chế độ Firebase** |
| Dữ liệu trên Firestore | ✅ Có đủ 4 collection `rca_records`, `rca_actions`, `rca_meta`, `rca_users` (người dùng đã đẩy dữ liệu lần đầu và tạo tài khoản). ⚠ **Chưa đối chiếu số lượng** RCA/action trên Firestore với dữ liệu gốc, chưa kiểm tra đồng bộ trên 2 máy thật |
| Mật khẩu chung (chỉ xem) | Người dùng đặt; do Admin quản lý ở tab 🛡 Người dùng. **Không ghi mật khẩu vào code hay tài liệu** (repo public) |
| Giao diện ILD Crafted | ↩ Đã thử (commit `de3c154`) rồi **hoàn tác** theo yêu cầu (commit `c2c6c5e`). Giao diện hiện tại là bản nâu/vàng gốc. Muốn áp lại: `git revert c2c6c5e` |

### Lịch sử commit chính

```
c2c6c5e Revert "new" — khôi phục giao diện trước ILD Crafted   ← HEAD
de3c154 new            (giao diện ILD Crafted — đã bị hoàn tác)
37ca6a5 người dùng     (tab Người dùng, hướng dẫn chế độ Firebase)
d2bd1b7 MK             (điền FIREBASE_CONFIG)
682cffe Firebase loading (đăng nhập + đồng bộ Firebase, firestore.rules)
3830b98 page           (đổi tên "RCA Tracking App.html" → index.html cho GitHub Pages)
85350fa Action         (cập nhật Status action 2 chiều)
e18a945 new RCA        (bản đầu)
```

---

## 3. Kiến trúc & luồng dữ liệu

```
 Firestore (nguồn sự thật dùng chung)
   rca_records/{id} · rca_actions/{id} · rca_meta/settings · rca_users/{uid}
        ▲  cloudPushNow(): chỉ ghi document thay đổi      │ onSnapshot (realtime)
        │  (so stableJson với CLOUD.known)                 ▼
   persist()  ◄──────────── state (biến JS toàn cục) ◄── cloudOnSnapshot() → persist(true) + cloudRenderAll()
        │                       │
        ▼                       ▼ render*() vẽ lại DOM thủ công (không framework)
 localStorage "rca_tracking_app_v1"  (bản sao offline)       UI
 + ghi RCA_Data_Backup.json ra thư mục đồng bộ (nếu đã chọn)
```

- `state` = `{rcaList[], actionList[], settings{}, rcaFilters, actFilters, rcaSort, actSort, expandedRca}`.
- **Mọi thay đổi dữ liệu đều đi qua `persist()`** → ghi localStorage → (nếu có) ghi file đồng bộ → `cloudSchedulePush()` (debounce 400 ms). `persist(true)` = chỉ ghi cache, không đẩy lên cloud (dùng khi vừa nhận dữ liệu từ cloud).
- Offline: Firestore `persistentLocalCache` tự xếp hàng ghi và gửi lại khi có mạng.
- Xung đột: **bản lưu sau thắng theo từng bản ghi** (không gộp 3 chiều).
- `FIREBASE_CONFIG = null` → **chế độ cũ (local)**: chỉ localStorage, Admin nhập mật khẩu `ADMIN_PASSWORD` (mặc định `QAILD`), Leader chọn tên không mật khẩu. Code chế độ cũ vẫn giữ nguyên để dùng khi cần.

### Bản đồ code (dòng ~, theo section comment)

| Dòng | Section | Nội dung |
|---|---|---|
| 8–446 | `<style>` | CSS; biến màu ở `:root` (sáng) + dark mode; cuối block là CSS màn hình đăng nhập / Người dùng |
| 486–880 | HTML các tab | DASHBOARD, RCA LIST, ACTION PLAN, USER GUIDE, DATA/SETTINGS, SETTINGS, USERS |
| 881–1166 | HTML modal | RCA, Action, Evidence, Login (chế độ cũ), Tài khoản, màn hình khoá `#authGate` |
| 1169 | SEED DATA | `SEED_DATA` — 64 RCA + 145 action import từ Excel (1 dòng JSON dài) |
| 1172 | CONFIG | Danh mục mặc định, `FUNC_DEPT_MAP`, `STORAGE_KEY`, `ADMIN_PASSWORD` |
| 1197 | STATE | `state`, `loadState()`, `persist()` |
| 1256 | AUTH | `currentUser`, `isAdmin/isLeader`, `isRelatedToRca`, `canEditRca/canEditAction…`, login chế độ cũ, `renderAuthBox`, `applyPermissionUI` |
| 1417 | CLOUD | `FIREBASE_CONFIG`, `CLOUD_OWNER_EMAIL`, `cloudInit`, `authLoadProfile`, `authApply`, màn hình đăng nhập, hộp Tài khoản, `cloudPushNow`, `cloudOnSnapshot`, `renderCloudStatus`, **quản lý người dùng** (`renderUsers`, `userCreate`…) |
| 2027 | UTIL | `uid`, `todayISO`, `fmtDate`, `escapeHtml`, `toast`, `effectiveRcaStatus`, `effectiveActionStatus`, `ACTION_STATUSES`, `applyActionStatus` |
| 2107 | EVIDENCE | Đính kèm file theo RCA (File System Access API) |
| 2380 | AUTO-SYNC | Ghi `RCA_Data_Backup.json` ra thư mục local mỗi lần `persist()` |
| 2522 | PRESENTATION MODE | A− / A+ / Đậm |
| 2564 | SETTINGS | Tab Cài đặt (Admin) |
| 2651 | TABS | Chuyển tab |
| 2663 | DROPDOWN POPULATION | Dropdown + multi-select filter |
| 2814 | DASHBOARD | KPI, biểu đồ (màu theo `STATUS_COLORS`) |
| 2922 | EMAIL REPORT | Sinh file `.eml` (biểu đồ PNG + bảng) |
| 3252 | RCA LIST | Lọc/sort/render bảng RCA |
| 3368 | ACTION PLAN LIST | Lọc/sort/render bảng action |
| 3448 | VIEW RCA DETAIL | Popup chi tiết RCA (có dropdown đổi Status action) |
| 3524 | RCA TEMPLATE REPORT | Xuất PDF/Excel theo biểu mẫu công ty, `RCA_LOGO_BASE64`, `getRcaTemplateData()` |
| 3729 | MODALS: RCA | Form RCA + checklist hành động (`buildInitialActionItems`, `withLiveStatus`, `syncActionItemsToActionPlan`) |
| 3939 | MODALS: ACTION | Form action |
| 4046 | FILTER WIRING | Ô tìm kiếm |
| 4057 | IMPORT / EXPORT | JSON backup/restore, CSV, khôi phục dữ liệu gốc |
| 4155 | INIT | `init()` |

---

## 4. Data model

### RCA (`state.rcaList[]` ↔ Firestore `rca_records/{id}`)

```
id, stt, rcaNumber, status, function, dept, leader, classification,
issueDate, completeDate, category, defectMode, issue, problemStatement,
rootCause, action, actionItems[], standards, fourM,
result1, result2, result3, evidence[]
```

- `action` (text) **tự sinh** từ `actionItems[]` khi lưu.
- `actionItems[]` = `{id, content, pic, dueDate}[]` — **`id` trùng `id` của action thật trong `actionList`** (liên kết 2 chiều). Status **không** lưu ở đây.
- `evidence[]` = metadata `{name, size, type, addedAt, addedBy}`; file thật nằm trên ổ đĩa máy người đính kèm.
- `status`: `On Progress | Late | Done | Cancel`; `Late` còn được suy ra tự động (`effectiveRcaStatus()`) theo `classification` + `settings.classDays`.

### Action (`state.actionList[]` ↔ `rca_actions/{id}`)

```
id, rcaNumber, issue, correctiveAction, pic, status, function, dept,
dueDate, completeDate, evidence, verifyEffective, remarks
```

- Nối với RCA qua **`rcaNumber`**.
- Action thuộc checklist của RCA (`isActionLinkedToRca()`): content/PIC/dueDate **khoá** ở form action, phải sửa từ form RCA.
- **Status cập nhật 2 chiều**: tab Kế hoạch hành động, dropdown trong popup chi tiết RCA (lưu ngay), cột Status trong checklist form RCA (lưu khi bấm Lưu). Nguồn sự thật duy nhất: `actionList[].status`. Chuyển sang `Done` mà chưa có Complete Date → tự điền hôm nay (`applyActionStatus()`).

### Settings (`state.settings` ↔ `rca_meta/settings`, chỉ Admin ghi)

```
functions[], depts[], categories[], classDays{Minor,Major,Serious}, leaders[], companyName
```

Document Firestore có thêm `_updatedAt`, `_updatedBy` (bỏ qua khi đọc — mọi field bắt đầu bằng `_`).

### Hồ sơ người dùng (`rca_users/{uid}`)

```
username, email, displayName, leaderName, role: viewer|leader|admin, active, shared, createdAt, createdBy
```

---

## 5. Đăng nhập & phân quyền (chế độ Firebase — đang dùng)

| Vai trò | Đăng nhập | Quyền |
|---|---|---|
| **Chỉ xem** | Màn hình khoá → mật khẩu chung (tài khoản Firebase `viewer@ild-rca.local`) | Xem, xuất báo cáo; không thêm/sửa |
| **Leader** | Tài khoản riêng `tên@ild-rca.local` (nút tên góc phải, hoặc "Đăng nhập tài khoản riêng") | Thêm RCA; sửa RCA/action có tên mình ở Leader hoặc PIC; đổi Status action |
| **Admin** | Tài khoản riêng vai trò Admin, hoặc Google `dangthanhbinh53@gmail.com` | Toàn quyền + tab ⚙️ Cài đặt + 🛡 Người dùng |

- `authApply()` ánh xạ hồ sơ → `currentUser = {role, name: leaderName}` để dùng lại toàn bộ `canEditRca/canEditAction…`. **`leaderName` phải trùng đúng tên trong cột Leader/PIC** của dữ liệu RCA.
- Tab 🛡 Người dùng: tạo/đổi mật khẩu chung, tạo tài khoản, đổi vai trò, sửa tên Leader, khoá/mở, đổi mật khẩu (Firebase gói Spark yêu cầu biết mật khẩu cũ), xoá quyền (tài khoản Auth vẫn còn — xoá hẳn ở Firebase Console). Tạo tài khoản dùng app Firebase phụ in-memory (`authSecondary`) để Admin không bị đăng xuất.
- **Quyền thật ở `firestore.rules`** (đã Publish): thành viên active đọc; Leader/Admin tạo/sửa RCA & action; chỉ Admin xoá, ghi `rca_meta`, quản lý `rca_users`.
- ⚠ Giới hạn: Rules **chưa** kiểm "Leader chỉ sửa RCA có tên mình" — chỉ giao diện chặn. Firebase Auth vẫn cho tự đăng ký tài khoản qua API, nhưng không có hồ sơ `rca_users` thì không đọc/ghi được gì.
- Sửa `firestore.rules` trong repo **không** tự áp dụng — phải dán vào Firebase Console → Firestore → Rules → Publish.

---

## 6. Tính năng theo tab

- **📊 Tổng quan** — KPI, biểu đồ trạng thái / Classification / Function / Category / tháng, action quá hạn, nút "Gửi email báo cáo" (tải `.eml`).
- **📋 Danh sách RCA** — multi-select filter 7 trường, sort mọi cột, tìm kiếm, click dòng xem chi tiết; nút 👁 / ✎ / +Act / 📎.
- **✅ Kế hoạch hành động** — bảng action, filter + sort.
- **📖 Hướng dẫn sử dụng** — hướng dẫn trong app; phần đăng nhập tự đổi nội dung theo chế độ (`.cloud-only` / `.local-only`).
- **🗂 Dữ liệu & Cấu hình** — trạng thái đồng bộ Firebase (+ nút đẩy dữ liệu lần đầu khi Firestore trống), sao lưu/phục hồi JSON, xuất CSV, đồng bộ tự động ra thư mục, thư mục Evidence.
- **⚙️ Cài đặt** (Admin) — tên công ty, hạn xử lý theo Classification, danh mục Function/Dept/Category/Leader, khôi phục dữ liệu gốc. (Ô đổi mật khẩu Admin cục bộ ẩn ở chế độ Firebase.)
- **🛡 Người dùng** (Admin, chỉ chế độ Firebase) — xem mục 5.
- **Xuất báo cáo 1 RCA** (popup chi tiết) — PDF (in) và Excel `.xls` (HTML giả dạng Excel — Excel cảnh báo định dạng, bấm Yes) theo biểu mẫu `RCA Template (new-25.12.2025).xlsx`.
- **A− / A+ / Đậm** — phóng to/in đậm khi trình chiếu.

---

## 7. Lưu trữ trên trình duyệt

| Key | Nội dung |
|---|---|
| `localStorage.rca_tracking_app_v1` | Bản sao offline `{rcaList, actionList, settings}` |
| `localStorage.rca_tracking_app_v1_auth_cache` | Hồ sơ đăng nhập gần nhất (để xem offline khi không tải được Firebase) |
| `localStorage.rca_tracking_app_v1_admin_password` | Chỉ chế độ cũ |
| `sessionStorage.rca_tracking_app_v1_session` | Chỉ chế độ cũ |
| `..._uiscale` / `..._uibold` / `..._email_from/to/cc` | Tuỳ chọn theo máy |
| IndexedDB `rca_evidence_fs_db` | Handle thư mục Evidence / thư mục đồng bộ |
| IndexedDB của Firebase | Phiên đăng nhập + cache Firestore offline |

File JSON sao lưu không chứa mật khẩu, cấu hình thư mục hay tài khoản.

---

## 8. Quyết định thiết kế đã chốt

1. Mục **"3. Phân tích 5WHY"** và bảng **"Correction tạm thời"** trong báo cáo cố ý để trống (app chưa có dữ liệu cấu trúc). Bổ sung = mở rộng data model, không phải sửa lỗi.
2. 4 ô **Safety/Quality/Volume/Cost** chỉ tích khi `category` khớp chính xác.
3. "Corrective actions" ở popup chi tiết và bảng trong báo cáo PDF/Excel **phải khớp 100%** — cùng lọc `actionList` theo `rcaNumber` (`getRcaTemplateData()`).
4. Status action chỉ có **một nguồn sự thật** (`actionList[].status`), không sao chép vào `actionItems`.
5. Giao diện giữ bản gốc (nâu/vàng); bản ILD Crafted đã bị người dùng yêu cầu hoàn tác.

---

## 9. Kiểm thử

- **Cú pháp**: lệnh node ở mục 0.
- **Firebase không cần project thật**: tạo bản sao `index.html` trong thư mục tạm, đổi `FIREBASE_CONFIG` thành `{apiKey:'fake',projectId:'rca-test'}` và `FIREBASE_SDK` thành `'./mock/'`, viết `mock/core.js` giả lập các hàm Firebase đang dùng (`initializeApp, getAuth, initializeAuth, onAuthStateChanged, signInWithEmailAndPassword, createUserWithEmailAndPassword, updatePassword, signOut, GoogleAuthProvider, signInWithPopup, initializeFirestore, persistentLocalCache, persistentMultipleTabManager, doc, collection, onSnapshot, getDoc, getDocs, setDoc, updateDoc, deleteDoc, writeBatch, waitForPendingWrites`) + 3 file `firebase-app.js / firebase-auth.js / firebase-firestore.js` chỉ `export * from './core.js'`, rồi phục vụ bằng `python -m http.server`. Cách này đã dùng để kiểm: đẩy dữ liệu lần đầu, viewer không ghi được, Leader chỉ sửa RCA của mình, đổi status chỉ ghi 1 document, nhận thay đổi từ máy khác, khoá tài khoản.
- **Chưa kiểm**: Firestore Rules bằng emulator; đồng bộ đồng thời trên 2 máy thật; số lượng bản ghi trên Firestore so với dữ liệu gốc.

---

## 10. Bảo trì thường gặp

- **Danh mục, hạn xử lý, tên công ty**: tab ⚙️ Cài đặt (không sửa code).
- **Tài khoản, mật khẩu chung**: tab 🛡 Người dùng. Quên mật khẩu → Firebase Console → Authentication → đặt lại, hoặc xoá quyền rồi tạo tài khoản tên khác.
- **Sửa quyền Firestore**: sửa `firestore.rules` rồi dán vào Console → Publish.
- **Logo báo cáo PDF/Excel**: base64 trong `RCA_LOGO_BASE64` — thay bằng script (`base64.b64encode`), không gõ tay.
- **Nạp lại dữ liệu gốc từ Excel mới**: đọc Excel bằng `openpyxl` → build JSON → thay `SEED_DATA`. Lưu ý: ở chế độ Firebase, dữ liệu thật nằm trên Firestore; `SEED_DATA` chỉ dùng khi bấm "Khôi phục dữ liệu gốc" (Admin) — thao tác này sẽ **ghi đè và xoá** dữ liệu trên Firestore.
- **Tắt Firebase, quay về chế độ cũ**: đặt `const FIREBASE_CONFIG = null;`.

---

## 11. Việc còn để ngỏ

- Đối chiếu số RCA/action trên Firestore với dữ liệu mong đợi; thử đồng bộ trên 2 máy.
- Rules chặn Leader sửa RCA không thuộc mình (cần lưu tên Leader/PIC trong hồ sơ để Rules so sánh).
- Màu biểu đồ (`STATUS_COLORS`), mẫu email `.eml` và báo cáo PDF/Excel vẫn dùng màu nâu cũ (nằm trong JS).
- Dữ liệu 5-Why có cấu trúc; phân loại hành động Tạm thời/Lâu dài.
- Chưa thử `.eml` trên new Outlook / Outlook Web / Mail (Mac).
- Repo public: toàn bộ `SEED_DATA` (dữ liệu RCA gốc) xem được trong mã nguồn trang. Nếu cần bảo mật, chuyển repo private (GitHub Pages với repo private cần gói trả phí) hoặc bỏ `SEED_DATA` khỏi file.

---

## 12. Cấu trúc thư mục

```
RCA/
├── index.html        app (toàn bộ)
├── firestore.rules   Firestore Security Rules (bản đã Publish lên project rca-tracking-ild)
└── HANDOFF.md        tài liệu này
```

`RCA_Data_Backup.json` (bản sao lưu cũ 10/09/2026) đã được xoá khỏi repo ngày 06/10/2026 vì dữ liệu chính nằm trên Firebase. Lấy lại nếu cần: `git checkout c2c6c5e -- RCA_Data_Backup.json`.
