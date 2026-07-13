# README — Kiến trúc & Bảo mật CRM (Team Nam VPS)

> File này dùng để đính kèm vào các phiên chat Claude mới khi cần sửa/thêm tính năng cho CRM,
> giúp Claude nắm ngay bối cảnh mà không cần giải thích lại từ đầu.

## 1. Thông tin chung
- **Web CRM đang chạy:** https://wondrous-mooncake-399304.netlify.app/
- **Repo GitHub:** https://github.com/lmhtnguyen6-dev/The-Duc-tao-crm-quan-ly-khach-hang- (nên để **Private**)
- **File nguồn duy nhất:** `index.html` (single-page app, HTML/JS thuần, không cần build)
- **Backend:** Firebase Realtime Database
  - `databaseURL`: `https://test-crm-1-35ae7-default-rtdb.asia-southeast1.firebasedatabase.app`
  - Project ID: `test-crm-1-35ae7`
- **Deploy:** Netlify, đã nối **Continuous Deployment** với GitHub — push lên nhánh `main` là tự động deploy, không cần kéo-thả thủ công.

## 2. Cấu trúc dữ liệu Firebase
```
/customers        → dữ liệu khách hàng (700+ record), có index theo idNhom, von
/history          → lịch sử chăm sóc khách hàng
/passwords/{id}   → mật khẩu member, LƯU DẠNG HASH SHA-256, id viết chữ THƯỜNG
/prefs            → tuỳ chọn cá nhân
```

## 3. Đăng nhập & phân quyền
- App dùng `signInAnonymously()` của Firebase Auth để thoả điều kiện `auth != null` trong Rules (đã bật Anonymous provider trong Firebase Console).
- Danh sách member khai báo trong code, biến `USERS`:
  ```js
  const USERS = [
    {name:'Nam',  id:'nam',  color:'#534AB7', role:'admin',  allow:null},
    {name:'Hiếu', id:'hieu', color:'#185FA5', role:'member', allow:null},
    {name:'Linh', id:'linh', color:'#0F6E56', role:'member', allow:null},
    {name:'Đức',  id:'duc',  color:'#854F0B', role:'member', allow:null},
  ];
  ```
  **⚠️ QUAN TRỌNG: KHÔNG bao giờ thêm field `pw:'...'` (mật khẩu dạng chữ thường) vào mảng này.**
  Mật khẩu thật chỉ được lưu ở Firebase `passwords/{id}` dưới dạng hash SHA-256.
- Hàm `verifyPw(u, input)` sẽ:
  1. Đọc `passwords/{u.id}` từ Firebase
  2. Nếu có giá trị → so sánh `sha256(input) === stored`
  3. Nếu Firebase lỗi/không đọc được → fallback so `input === u.pw` (hiện `u.pw` là `undefined` nên luôn sai — đây là chủ đích, không phải bug)
  4. **`id` trong `passwords` phải viết chữ THƯỜNG khớp chính xác với `id` trong `USERS`** (Firebase phân biệt hoa/thường) — lỗi từng gặp: key `Duc`/`Linh`/`Nam` viết hoa chữ đầu khiến chỉ mình `hieu` đăng nhập được.

## 4. Cách đổi/thêm mật khẩu member
Dùng công cụ HTML độc lập `tao-hash-mat-khau.html` (đã tạo sẵn, chạy offline trong trình duyệt, không gửi dữ liệu đi đâu):
1. Mở file, nhập `id` (chữ thường) + mật khẩu mới
2. Bấm "Tạo Hash" → copy chuỗi hash
3. Vào Firebase Console → Realtime Database → node `passwords` → thêm/sửa key đúng `id` đó → dán hash vào

## 5. Firebase Realtime Database Rules (hiện tại)
```json
{
  "rules": {
    ".read": "auth != null",
    ".write": "auth != null",
    "customers": {
      ".indexOn": ["idNhom", "von"]
    },
    "passwords": {
      "$uid": {
        ".validate": "newData.isString()"
      }
    }
  }
}
```
→ Chỉ user đã qua `signInAnonymously()` mới đọc/ghi được. Đây là lớp bảo vệ chính vì API key Firebase vốn không phải bí mật tuyệt đối (bắt buộc lộ ra phía client).

## 6. Google Cloud API Key Restriction
- Key: "Browser key (auto created by Firebase)" trong Google Cloud Console → APIs & Services → Credentials
- Application restrictions: **Websites**, giới hạn theo domain:
  ```
  wondrous-mooncake-399304.netlify.app/*
  localhost/*
  ```
- Nếu đổi sang domain riêng sau này, phải quay lại thêm domain mới vào đây.

## 7. Sự cố đã xử lý (lịch sử, để tránh lặp lại)
1. **Lộ mật khẩu plain text lên GitHub** — mảng `USERS` từng có field `pw:'Nam2506'` v.v. GitHub Secret Scanning đã cảnh báo lộ Google API Key kèm theo. Đã xử lý: xoá `pw` khỏi code, đổi toàn bộ mật khẩu cũ, giới hạn API key theo domain.
2. **Sai case-sensitive giữa `USERS.id` và `passwords/{key}`** khiến chỉ 1 member đăng nhập được trong khi các member khác báo sai mật khẩu — do gõ hoa chữ đầu khi tạo key trong Firebase Console. Luôn dùng chữ thường.
3. Lịch sử Git cũ (2 commit đầu) vẫn còn lưu mật khẩu cũ dạng plain text vĩnh viễn — không xoá được, chỉ có thể hạn chế bằng cách chuyển repo sang Private. Mật khẩu cũ đó đã bị vô hiệu hoá do đã đổi sang giá trị mới.

## 8. Quy trình sửa/thêm tính năng (để đỡ tốn token ở các phiên chat mới)
1. Mở chat Claude **mới**
2. Đính kèm file này + file `index.html` mới nhất (tải từ GitHub, nút "Raw")
3. Mô tả tính năng cần thêm/sửa
4. Nhận code đã sửa → dán đè vào `index.html` local (thư mục đã clone bằng GitHub Desktop)
5. GitHub Desktop: Commit → Push origin
6. Netlify tự động deploy (đã bật Continuous Deployment, auto-publish nhánh `main`)
7. Kiểm tra lại web bằng hard refresh (Ctrl+Shift+R) hoặc cửa sổ ẩn danh
