# SRS – Đặc tả Yêu cầu Phần mềm: Chức năng Đăng nhập (Login)

| Mục | Giá trị |
|---|---|
| **Hệ thống** | Perfex CRM – Anh Tester Demo |
| **Chức năng** | Đăng nhập khu quản trị (Admin Authentication) |
| **URL khảo sát** | `https://crm.anhtester.com/admin/authentication` |
| **Tài khoản dùng khảo sát** | `admin@example.com` / `123456` (đăng nhập thành công, chuyển tới `/admin/`) |
| **Phương pháp** | Khảo sát UI thực tế + đọc DOM + quan sát điều hướng (in-app browser) |
| **Ngày khảo sát** | 2026-08-27 |
| **Trình duyệt** | Chromium, viewport 1280×720 |
| **Phiên bản tài liệu** | 1.0 |

---

## 1. Giới thiệu

### 1.1. Mục đích

Tài liệu này đặc tả yêu cầu phần mềm cho **chức năng Đăng nhập** của khu quản trị Perfex CRM.
Nó là cơ sở để thiết kế test case, kiểm thử hồi quy và đánh giá nghiệm thu chức năng.

### 1.2. Phạm vi

**Trong phạm vi**

- Màn hình đăng nhập `/admin/authentication`: bố cục, các trường nhập, nút, liên kết.
- Xác thực bằng cặp **Email + Mật khẩu**.
- Kiểm tra dữ liệu đầu vào (bắt buộc, định dạng email) và xử lý lỗi.
- Tuỳ chọn **Remember me** (ghi nhớ đăng nhập).
- Bảo vệ CSRF trên biểu mẫu đăng nhập.
- Điều hướng sau khi đăng nhập thành công / thất bại.
- Bảo vệ URL nội bộ khi chưa đăng nhập.

**Ngoài phạm vi**

- Luồng Quên mật khẩu / Đặt lại mật khẩu (chỉ nêu liên kết).
- Luồng Đăng xuất.
- Đăng nhập cổng khách hàng (front-end ngoài `/admin`).
- Phân quyền theo vai trò sau khi đăng nhập.
- Kiểm chứng thời gian sống của phiên.

### 1.3. Định nghĩa & viết tắt

| Thuật ngữ | Ý nghĩa |
|---|---|
| SRS | Software Requirements Specification |
| CSRF | Cross-Site Request Forgery |
| AC | Acceptance Criteria – tiêu chí chấp nhận |
| Khu quản trị | Vùng ứng dụng dưới đường dẫn `/admin` |
| Phiên | Trạng thái đã xác thực của người dùng trên máy chủ |

### 1.4. Tài liệu tham khảo

- Khảo sát UI thực tế trang `https://crm.anhtester.com/admin/authentication` ngày 2026-08-27.
- Tài liệu liên quan trong dự án: `requirements_login/requirements_login.md`, `requirements_login/TestCase_Login.md`.

---

## 2. Mô tả tổng quan

### 2.1. Bối cảnh sản phẩm

Trang đăng nhập là **cổng vào duy nhất** của khu quản trị `/admin`. Mọi màn hình nghiệp vụ khác đều
nằm sau lớp xác thực này. Khi chưa đăng nhập, truy cập bất kỳ URL nội bộ nào đều bị đưa về trang đăng nhập.

### 2.2. Chức năng chính

1. Hiển thị biểu mẫu đăng nhập.
2. Tiếp nhận và kiểm tra Email, Mật khẩu.
3. Xác thực thông tin đăng nhập với máy chủ.
4. Tạo phiên và điều hướng vào Dashboard khi hợp lệ.
5. Hiển thị thông báo lỗi và giữ người dùng ở lại trang khi không hợp lệ.
6. Phát hành cookie ghi nhớ khi người dùng chọn **Remember me**.
7. Bảo vệ biểu mẫu bằng mã CSRF.

### 2.3. Đặc điểm người dùng

| Nhóm | Mô tả |
|---|---|
| Nhân viên nội bộ (Admin, Project Manager…) | Có tài khoản khu `/admin`, dùng trang này để vào hệ thống |
| Khách chưa đăng nhập | Chỉ xem được trang đăng nhập / quên mật khẩu |
| Tài khoản cổng khách hàng | Bị từ chối ở khu `/admin`, không tạo được phiên quản trị |

### 2.4. Ràng buộc & Giả định

- Trang **không nạp tệp JavaScript nào** (0 thẻ `<script>`); toàn bộ kiểm tra dữ liệu bắt buộc chạy ở **máy chủ**.
- Kiểm tra định dạng email được thực hiện bởi **trình duyệt** thông qua `input[type=email]`; nội dung thông báo phụ thuộc trình duyệt.
- Thông báo lỗi hiển thị bằng **tiếng Anh**.
- Môi trường là bản demo dùng chung; mọi thao tác kiểm thử chỉ đọc hoặc hoàn tác được.

---

## 3. Đặc tả giao diện

### 3.1. Giao diện người dùng – Màn hình đăng nhập

| Thành phần | Loại | Thuộc tính quan sát được |
|---|---|---|
| Logo Anh Tester | Ảnh, có liên kết | Bọc trong `<a href="https://crm.anhtester.com/">` |
| Tiêu đề "Login" | Heading | — |
| **Email Address** | `input#email[type=email]` | `name=email`, `autofocus`, không có `required`/`maxlength`/`pattern` trong HTML |
| **Password** | `input#password[type=password]` | `name=password`, hiển thị dạng chấm che, không có nút hiện/ẩn |
| **Remember me** | `input#remember[type=checkbox]` | `name=remember`, `value=estimate`, mặc định **không tích** |
| (ẩn) CSRF token | `input[type=hidden]` | `name=csrf_token_name`, giá trị **32 ký tự hex**, đổi theo phiên |
| **Login** | `button[type=submit]` | Nhãn "Login", class `btn btn-primary btn-block`, luôn ở trạng thái bật |
| **Forgot Password?** | Liên kết | Trỏ tới `/admin/authentication/forgot_password` |

- Biểu mẫu gửi `POST` về chính `https://crm.anhtester.com/admin/authentication`.
- Toàn trang chỉ có **1** `<form>`, **1** `<button type=submit>`, **2** thẻ `<a>` (logo + Forgot Password).
- Không có CAPTCHA, không có nút đăng nhập mạng xã hội / OAuth.
- Tiêu đề tab: `Perfex CRM | Anh Tester Demo - Login`; `body` mang class `login_admin`.

### 3.2. Giao diện phần cứng / phần mềm

- Chạy trên trình duyệt web hiện đại có hỗ trợ HTML5 (`type=email`).
- Giao tiếp với máy chủ qua HTTPS.

### 3.3. Giao diện truyền thông

| Yêu cầu | Phương thức | Điểm cuối | Tham số thân |
|---|---|---|---|
| Nạp trang đăng nhập | `GET` | `/admin/authentication` | — |
| Gửi đăng nhập | `POST` | `/admin/authentication` | `csrf_token_name`, `email`, `password`, `remember` (khi tích) |

---

## 4. Đặc tả trường dữ liệu

### 4.1. Biểu mẫu đăng nhập – `/admin/authentication`

| Trường (Label) | Tên field | Bắt buộc | Ràng buộc | Mặc định | Ghi chú |
|---|---|---|---|---|---|
| Email Address | `email` | **Có** – kiểm ở máy chủ | Không có `required`/`maxlength`/`pattern` trong HTML. Định dạng do `type=email` của trình duyệt kiểm. Không phân biệt hoa/thường; bỏ qua khoảng trắng đầu/cuối | Rỗng | Có `autofocus`. Không có `autocomplete` |
| Password | `password` | **Có** – kiểm ở máy chủ | Không có ràng buộc độ dài / độ mạnh ở màn hình này | Rỗng | Hiển thị che ký tự. Không có nút hiện/ẩn |
| Remember me | `remember` | Không | Checkbox; giá trị gửi lên khi tích là chuỗi `estimate` | Không tích | Giá trị `estimate` không mang nghĩa nghiệp vụ ở màn hình này |
| csrf_token_name | `csrf_token_name` | Có – hệ thống tự điền | Chuỗi 32 ký tự hex, đổi theo phiên | Tự sinh | Bắt buộc cho mọi `POST` |

---

## 5. Yêu cầu chức năng

> Quy ước: mỗi yêu cầu có mã `FR-LOGIN-xx`, mô tả và **Acceptance Criteria**. Trạng thái:
> 🟢 = đã kiểm chứng thực tế trong đợt khảo sát · 🟡 = quan sát UI/DOM, chưa tương tác đầy đủ.

### 5.1. Hiển thị trang đăng nhập

| Mã | Yêu cầu | Acceptance Criteria | Trạng thái |
|---|---|---|---|
| FR-LOGIN-01 | Truy cập trang đăng nhập | Người dùng chưa đăng nhập mở `/admin/authentication` → trang trả HTTP 200, tiêu đề tab `Perfex CRM \| Anh Tester Demo - Login`, tiêu đề trang `Login`, `body` có class `login_admin` | 🟢 |
| FR-LOGIN-02 | Thành phần biểu mẫu | Biểu mẫu gồm đúng: `#email[type=email]`, `#password[type=password]`, `#remember[type=checkbox]`, nút `Login[type=submit]`, liên kết `Forgot Password?`, và 1 field ẩn `csrf_token_name`. Biểu mẫu `POST` về `/admin/authentication` | 🟢 |
| FR-LOGIN-03 | Tự đặt con trỏ vào ô Email | `#email` có thuộc tính `autofocus`; khi mở trang, con trỏ nằm sẵn ở ô Email | 🟢 |
| FR-LOGIN-04 | Logo dẫn về trang chủ | Liên kết bọc logo trỏ tới `https://crm.anhtester.com/` | 🟢 |
| FR-LOGIN-05 | Không có CAPTCHA / đăng nhập bên thứ ba | Không tồn tại phần tử CAPTCHA; không có nút đăng nhập Google/Facebook/OAuth; toàn trang chỉ có 1 `<form>` và 1 `<button type=submit>` | 🟢 |
| FR-LOGIN-06 | Trang không phụ thuộc JavaScript | Đếm được **0** thẻ `<script>` (inline và `src`) trên trang | 🟢 |

### 5.2. Đăng nhập thành công

| Mã | Yêu cầu | Acceptance Criteria | Trạng thái |
|---|---|---|---|
| FR-LOGIN-07 | Đăng nhập với thông tin hợp lệ | Nhập `admin@example.com` + `123456` → sau khi bấm Login, trình duyệt dừng ở `https://crm.anhtester.com/admin/`, tiêu đề tab chứa chuỗi `Dashboard` | 🟢 |
| FR-LOGIN-08 | Email không phân biệt hoa thường | Nhập email đúng nhưng khác kiểu chữ (ví dụ `ADMIN@Example.COM`) + mật khẩu đúng → đăng nhập thành công, chuyển tới `/admin/` | 🟡 |
| FR-LOGIN-09 | Bỏ qua khoảng trắng thừa đầu/cuối email | Nhập `  admin@example.com  ` + mật khẩu đúng → đăng nhập thành công | 🟡 |
| FR-LOGIN-10 | Ghi nhớ đăng nhập phát hành cookie | Tích **Remember me** trước khi đăng nhập → hệ thống phát hành cookie `autologin` (chuỗi PHP-serialize gồm `user_id` và `key` 16 ký tự hex) | 🟡 |
| FR-LOGIN-11 | Không tích Remember me thì không có cookie ghi nhớ | Bỏ trống **Remember me** khi đăng nhập → sau khi vào `/admin/`, không có cookie `autologin` | 🟡 |

### 5.3. Kiểm tra dữ liệu đầu vào & đăng nhập thất bại

| Mã | Yêu cầu | Acceptance Criteria | Trạng thái |
|---|---|---|---|
| FR-LOGIN-12 | Bỏ trống cả hai trường | Gửi biểu mẫu rỗng → trang nạp lại, hiển thị **2** banner `.alert.alert-danger` theo thứ tự: `The Password field is required.` rồi `The Email Address field is required.` | 🟢 |
| FR-LOGIN-13 | Bỏ trống riêng Email | Nhập mật khẩu, để trống email → hiển thị **1** banner: `The Email Address field is required.` | 🟡 |
| FR-LOGIN-14 | Bỏ trống riêng Mật khẩu | Nhập email, để trống mật khẩu → hiển thị **1** banner: `The Password field is required.` | 🟡 |
| FR-LOGIN-15 | Chặn email sai định dạng tại trình duyệt | Nhập email thiếu ký tự `@` (ví dụ `notanemail`) rồi bấm Login → trình duyệt chặn gửi biểu mẫu (`#email.checkValidity() === false`), không phát sinh request POST, trang không nạp lại. Nội dung tooltip do trình duyệt sinh, **không** dùng làm assertion | 🟢 |
| FR-LOGIN-16 | Thông báo khi sai thông tin đăng nhập | Nhập email đúng + mật khẩu sai (ví dụ `admin@example.com` + `wrongpass`) → hiển thị **1** banner `.alert.alert-danger`: `Invalid email or password`; người dùng vẫn ở `/admin/authentication` | 🟢 |
| FR-LOGIN-17 | Thông báo lỗi không tiết lộ email nào có thật | Email không tồn tại + mật khẩu bất kỳ, và email có thật + mật khẩu sai → **cùng một chuỗi** `Invalid email or password`, cùng số lượng banner, cùng URL | 🟡 |
| FR-LOGIN-18 | Không khoá tài khoản sau nhiều lần sai | Đăng nhập sai mật khẩu nhiều lần liên tiếp với cùng email → mọi lần đều nhận `Invalid email or password`, không có thông báo khoá; ngay sau đó đăng nhập bằng mật khẩu đúng vẫn thành công | 🟡 |
| FR-LOGIN-19 | Giữ lại email đã nhập sau khi đăng nhập thất bại | Sau lỗi đăng nhập, HTML máy chủ trả về nên điền sẵn `value` của `#email` bằng email vừa nhập. *(Hiện trạng khảo sát trước đây ghi nhận máy chủ KHÔNG trả lại giá trị – cần xác nhận lại và mở lỗi nếu vẫn còn.)* | 🟡 |

### 5.4. Bảo vệ phiên, CSRF & điều hướng

| Mã | Yêu cầu | Acceptance Criteria | Trạng thái |
|---|---|---|---|
| FR-LOGIN-20 | Chặn URL nội bộ khi chưa đăng nhập | Mở thẳng một URL trong `/admin` (ví dụ `/admin/clients`) khi chưa đăng nhập → bị chuyển hướng, trình duyệt dừng ở `/admin/authentication` | 🟡 |
| FR-LOGIN-21 | Không ghi nhớ URL đích | URL sau chuyển hướng là `/admin/authentication` trần – không có tham số `?redirect=` / `?return_url=`. Sau khi đăng nhập, vào Dashboard chứ không quay lại URL ban đầu | 🟡 |
| FR-LOGIN-22 | Đã đăng nhập thì không vào lại trang đăng nhập | Mở `/admin/authentication` khi đang có phiên → bị đưa về `/admin/`, tiêu đề tab chứa `Dashboard` | 🟢 |
| FR-LOGIN-23 | Biểu mẫu mang mã chống CSRF | Mỗi lần nạp trang, biểu mẫu chứa `input[type=hidden][name=csrf_token_name]` với giá trị 32 ký tự hex; thân `POST` chứa `csrf_token_name=<token>` | 🟢 |
| FR-LOGIN-24 | Từ chối yêu cầu có mã CSRF sai | Gửi biểu mẫu với `csrf_token_name` bị sửa → máy chủ trả HTTP 403, tiêu đề tab `Error`, nội dung `419 Page Expired!` / `Sorry, the page has expired, return to previous page and refresh to continue.` | 🟡 |
| FR-LOGIN-25 | Tài khoản cổng khách hàng bị từ chối ở khu quản trị | Gửi biểu mẫu `/admin/authentication` bằng tài khoản cổng khách hàng hợp lệ → hiển thị **1** banner `Invalid email or password`, không tạo phiên quản trị | 🟡 |

---

## 6. Business Rules & Thông báo (nguyên văn từ UI)

> Thông báo hiển thị trong khung `.alert.alert-danger.text-center` đặt phía trên các trường nhập.

| Tình huống | Thông báo mong đợi (nguyên văn) |
|---|---|
| Bỏ trống cả Email và Password | `The Password field is required.` (banner 1) và `The Email Address field is required.` (banner 2) |
| Bỏ trống Email, có Password | `The Email Address field is required.` |
| Có Email, bỏ trống Password | `The Password field is required.` |
| Email thiếu ký tự `@` | Tooltip do **trình duyệt** sinh (đổi theo trình duyệt/ngôn ngữ). Chỉ assert `checkValidity() === false` |
| Email không tồn tại **hoặc** mật khẩu sai | `Invalid email or password` |
| Mã CSRF không hợp lệ | Trang lỗi HTTP 403 – `419 Page Expired!` / `Sorry, the page has expired, return to previous page and refresh to continue.` |

**Quy tắc nghiệp vụ**

- BR-01: Email được chuẩn hoá trước khi so khớp – không phân biệt hoa/thường, cắt khoảng trắng đầu/cuối.
- BR-02: Thông báo đăng nhập thất bại là **thông báo chung**, không phân biệt "email không tồn tại" với "sai mật khẩu".
- BR-03: Kiểm tra "trường bắt buộc" chạy **trước** khi kiểm tra thông tin đăng nhập; nếu thiếu trường thì không truy vấn xác thực.
- BR-04: Mọi `POST` tới `/admin/authentication` phải kèm mã CSRF hợp lệ.
- BR-05: Không có cơ chế khoá tài khoản / giới hạn số lần thử theo email hoặc IP (theo khảo sát hiện tại).

---

## 7. Luồng xử lý

### 7.1. Đăng nhập thành công

```
1. Mở /admin/authentication              → biểu mẫu hiện ra, con trỏ ở ô Email
2. Nhập Email Address hợp lệ (admin@example.com)
3. Nhập Password đúng (123456)
4. (tuỳ chọn) Tích Remember me            → sẽ nhận cookie autologin
5. Bấm Login                              → POST /admin/authentication (kèm csrf_token_name)
6. Máy chủ xác thực hợp lệ → tạo phiên → chuyển hướng
7. Trình duyệt dừng ở /admin/ (Dashboard), tiêu đề tab chứa "Dashboard"
```

### 7.2. Đăng nhập thất bại (sai thông tin)

```
1..5. Như trên nhưng Email hoặc Password sai
6. Máy chủ từ chối → nạp lại /admin/authentication
7. Hiển thị banner "Invalid email or password"; người dùng ở lại trang
```

### 7.3. Thiếu trường bắt buộc

```
1. Bỏ trống Email và/hoặc Password → bấm Login
2. Máy chủ nạp lại trang, hiển thị banner "The <Field> field is required."
   cho từng trường còn thiếu; không thực hiện xác thực
```

### 7.4. Email sai định dạng

```
1. Nhập chuỗi không phải email (thiếu "@") vào ô Email → bấm Login
2. Trình duyệt chặn submit (validation HTML5); không có request tới máy chủ
```

---

## 8. Ma trận trạng thái phiên

| Trạng thái hiện tại | Hành động | Trạng thái kế tiếp |
|---|---|---|
| Chưa đăng nhập | Gửi biểu mẫu với thông tin đúng | Đã đăng nhập (ở `/admin/`) |
| Chưa đăng nhập | Gửi biểu mẫu với thông tin sai / thiếu | Chưa đăng nhập (hiện thông báo lỗi) |
| Chưa đăng nhập | Mở URL nội bộ trong `/admin` | Chưa đăng nhập (bị đưa về trang đăng nhập) |
| Đã đăng nhập | Mở lại `/admin/authentication` | Đã đăng nhập (bị đưa về Dashboard) |

---

## 9. Yêu cầu phi chức năng

| Mã | Loại | Yêu cầu |
|---|---|---|
| NFR-01 | Bảo mật | Biểu mẫu đăng nhập phải có mã CSRF hợp lệ cho mỗi phiên; yêu cầu sai mã bị từ chối (HTTP 403). |
| NFR-02 | Bảo mật | Thông báo lỗi đăng nhập không được tiết lộ email nào tồn tại trong hệ thống. |
| NFR-03 | Bảo mật | Trường mật khẩu phải hiển thị che ký tự. |
| NFR-04 | Bảo mật | Cookie phiên của ứng dụng phải có cờ `HttpOnly` (không đọc được bằng `document.cookie`). |
| NFR-05 | Khả dụng | Ô Email được `autofocus` khi mở trang. |
| NFR-06 | Khả dụng | Thông báo lỗi hiển thị rõ ràng phía trên biểu mẫu, giữ nguyên trên cùng trang. |
| NFR-07 | Tương thích | Trang hoạt động không cần JavaScript; kiểm tra bắt buộc thực hiện ở máy chủ. |
| NFR-08 | Hiệu năng | Trang đăng nhập trả về HTTP 200 và hiển thị đầy đủ trong thời gian hợp lý (< 3s ở điều kiện mạng bình thường). |
| NFR-09 | Truyền tải | Toàn bộ giao tiếp qua HTTPS; không đặt email/mật khẩu trên query string. |

---

## 10. Vấn đề mở / Cần xác nhận

| Mã | Nội dung |
|---|---|
| OPEN-01 | Thời gian sống của phiên và điều kiện tự hết hạn chưa được kiểm chứng. |
| OPEN-02 | Hành vi thực tế của cookie `autologin` (tự đăng nhập lại) chưa kiểm chứng trong đợt này; tài liệu dự án ghi nhận tính năng "Remember me" có thể không hoạt động. |
| OPEN-03 | Việc máy chủ có giữ lại giá trị email sau khi đăng nhập thất bại (FR-LOGIN-19) cần kiểm chứng lại; hồ sơ trước đây ghi nhận là lỗi. |
| OPEN-04 | Giá trị `value=estimate` của checkbox Remember me không rõ ý nghĩa nghiệp vụ. |
| OPEN-05 | Nội dung/định dạng thông báo đăng nhập chỉ có tiếng Anh; chưa rõ yêu cầu đa ngôn ngữ. |

---

## 11. Lịch sử sửa đổi

| Phiên bản | Ngày | Người thực hiện | Ghi chú |
|---|---|---|---|
| 1.0 | 2026-08-27 | Khảo sát tự động (Claude Code) | Bản đầu tiên – dựng từ khảo sát UI thực tế trang đăng nhập với tài khoản `admin@example.com`. |
