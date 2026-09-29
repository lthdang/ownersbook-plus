# HASHST_DEV-4556 Các bước kiểm tra local (Evidence đính kèm MR)

Đối tượng: Thay đổi cho phép chữ Hán (Kanji) nằm ngoài phạm vi trong `verifyInputText` (kiểm tra loại ký tự nhập vào / 入力文字種チェック) của app-vuejs
Branch: `backlog/HASHST_DEV-4556` (So sánh với: `f68dad7d` = `origin/js-ph2`)

Trong MR, đính kèm ảnh chụp màn hình (screenshot) của **Trước khi sửa** (`f68dad7d`) và **Sau khi sửa** (`backlog/HASHST_DEV-4556`) cho từng case.

---

## 0. Chia sẻ trước: DB local không thể lưu ký tự「𠮷」(Đã xác nhận)

Đã xác nhận các điểm sau trên MySQL local (`st-mysql`):

- Các cột text trong schema như `hhd` / `basis` **đều là `utf8mb3` (3 byte)**. Các cột trong thông tin khách hàng như `NAME`, `ADDRESS1〜4`, `EMPLOYMENT` của `ECO_USER`・`ECO_USER_CHANGE`・`ECO_USER_REQUEST` cũng tương tự
- `sql_mode` có `STRICT_TRANS_TABLES`, và kết nối DB của common-front-api cũng là `strict => true`
- Kết quả đã kiểm tra trên bảng tạm (cột `utf8mb3`):
  - Có thể lưu「山﨑」「㐂」(3 byte)
  - 「𠮷田」(4 byte) bị lỗi `ERROR 1366 Incorrect string value`

Vì vậy, tại các màn hình không gọi `checkJpText` (kiểm tra SJIS-win), nếu nhập「𠮷」và gửi (submit) thì **dự kiến sẽ bị lỗi DB ở phía server**. Trước khi sửa, xử lý bị chặn lại bởi lỗi định dạng (format error) ở frontend, do đó cách xuất hiện lỗi sẽ thay đổi. Bắt buộc phải ghi lại kết quả trong **Case B-1** bên dưới (tương ứng với nội dung "nếu ký tự 4 byte không thể lưu được thì sẽ xem xét lại" trong ADR 0002).

---

## 1. Chuẩn bị

### 1-1. Môi trường

1. Khởi động các container trong local-docker (`st-mysql`, `st-web`, common-front-api / passport / st-front-api)
2. Xác nhận file `.env` của app-vuejs đang trỏ tới API local (`VUE_APP_COMMON_API` / `VUE_APP_PASSPORT_API` / `VUE_APP_FUND_API` có giá trị là `local-*`)
3. Khởi động bằng `npm run dev` (port 8080). Khi gặp lỗi OpenSSL, thêm `NODE_OPTIONS=--openssl-legacy-provider`
4. Mở trình duyệt bằng Chrome ở chế độ **giả lập thiết bị (smartphone / デバイスエミュレーション)**
   - Nếu dùng User-Agent (UA) của PC, các màn hình Họ và tên (お名前) (l2212)・Quốc tịch (国籍) (l2213) sẽ bị điều hướng sang màn hình mã QR (`/l2325`)
   - Nút gửi (submit) của l2212・l2213 cũng sẽ hiển thị alert cảnh báo dành cho PC

### 1-2. Chuyển đổi giữa Trước khi sửa và Sau khi sửa

```bash
# Trước khi sửa (nguồn so sánh)
git switch --detach f68dad7d
npm run dev

# Sau khi sửa
git switch backlog/HASHST_DEV-4556
npm run dev
```

Các thay đổi dành cho local trong `config/dev.env.js` (chưa commit) vẫn sẽ được giữ nguyên. Không commit file này.

### 1-3. Tài khoản test

| Mục đích | Nội dung |
| --- | --- |
| Cá nhân (個人) | Có thể đăng nhập, và thông tin khách hàng không ở trạng thái "Đang thay đổi" (「変更中」) (các mục "Đang thay đổi" sẽ không click được) |
| Pháp nhân (法人) | Tương tự như trên. Có thể vào màn hình `/l2900` qua Cài đặt (設定) → Xác nhận / Thay đổi thông tin khách hàng (お客様情報の確認・変更) |
| Nhóm xác minh danh tính (本人確認系) (Tùy chọn) | Tài khoản cá nhân có họ tên đăng ký chứa chữ「﨑」(chuẩn bị trong test data của DB local). Không thể chuẩn bị ký tự「𠮷」do không lưu được vào DB |

Sử dụng giá trị giả lập cho test data. Không đưa thông tin của khách hàng thực tế vào màn hình hoặc screenshot.

### 1-4. Các ký tự cần nhập

| Giá trị nhập | Ký tự | Dự kiến kết quả qua 3 tầng check |
| --- | --- | --- |
| `髙橋` | 髙 U+9AD9 (nằm trong phạm vi, dùng làm đối chứng) | Cả trước và sau khi sửa đều pass |
| `山﨑` | 﨑 U+FA11 (chữ Hán tương thích CJK / CJK 互換漢字, 3 byte) | ① Sau khi sửa sẽ pass → ② SJIS-win hợp lệ → DB lưu được |
| `㐂` | U+3402 (Mở rộng A / 拡張 A, 3 byte) | ① Sau khi sửa sẽ pass → ② SJIS-win **không hợp lệ** → DB lưu được |
| `𠮷田` | 𠮷 U+20BB7 (Mở rộng B / 拡張 B, 4 byte) | ① Sau khi sửa sẽ pass → ② SJIS-win **không hợp lệ** → DB **không lưu được** |
| `山田😀` | Emoji U+1F600 | ① Cả trước và sau khi sửa đều **bị chặn** |

Vì các ký tự「㐂」「𠮷」khó gõ bằng IME, hãy copy và paste từ bảng này.

### 1-5. Thông điệp lỗi (Error message)

- **Lỗi định dạng (Format error)**: Khác nhau tùy theo từng màn hình (được ghi rõ trong từng case dưới đây). Hiển thị bằng chữ màu đỏ bên dưới ô nhập
- **Lỗi SJIS-win** (`checkJpText`, E103-1045):「{文字}の入力はできません。ひらがなで入力してください。」(Không thể nhập {ký tự}. Vui lòng nhập bằng hiragana.). Xuất hiện khi unfocus khỏi ô nhập (blur). Nếu ô đó đang có lỗi định dạng thì hàm này sẽ không được gọi

---

## 2. Các case kiểm tra

Mức độ ưu tiên **A** là bắt buộc. Mức **B** là các màn hình có cơ chế tương tự như A, nên sẽ kiểm tra nếu còn thời gian.

Tên file screenshot đặt theo định dạng `<Mã_case>_<before|after>_<Giá_trị_nhập>.png` (ví dụ: `A-1_after_山﨑.png`).

### A-1. Cá nhân: Họ và tên (お名前) (l2212) — Có check định dạng ＋ check SJIS-win

**Các bước thực hiện**

1. Đăng nhập bằng tài khoản cá nhân → My Page (マイページ) → Cài đặt (設定) (`/l2100`) → "Xác nhận / Thay đổi thông tin khách hàng" (お客様情報の確認・変更) → Nhập mã PIN giao dịch (取引暗証番号) → `/l2200`
2. Tại card "Họ và tên" (お名前), chọn "Thay đổi" (変更) → `/l2212`. Chọn "Thay đổi thông tin" (情報の変更)
3. Nhập các giá trị theo bảng bên dưới vào ô **Họ và tên (toàn giác)「Họ」** (お名前（全角）「姓」), rồi click vào ô「Tên」(名) để bỏ focus (blur)
4. Chụp lại màn hình hiển thị

| Nhập (Họ) | Trước khi sửa | Sau khi sửa |
| --- | --- | --- |
| `髙橋` | Không có lỗi | Không có lỗi |
| `山﨑` | 「姓を正しく入力してください。」 | **Không có lỗi** |
| `㐂` | 「姓を正しく入力してください。」 | 「㐂の入力はできません。ひらがなで入力してください。」 |
| `𠮷田` | 「姓を正しく入力してください。」 | 「𠮷の入力はできません。ひらがなで入力してください。」 |
| `山田😀` | 「姓を正しく入力してください。」 | 「姓を正しく入力してください。」 |

5. Với địa chỉ, tại các mục **Quận/Huyện/Xã/Phường** (市区町村・町域名) (type3)・**Số nhà/Ngõ ngách** (番地) (type7)・**Tên tòa nhà/Số phòng** (建物名・部屋番号等) (type3), cũng nhập `山﨑町` / `㐂` / `𠮷田` và kiểm tra tương tự
   - Lỗi định dạng trước khi sửa:「漢字/ひらがな/全角カタカナ/数字/英字/ー/-/空白スペースを入力してください。」(Vui lòng nhập Hán tự/Hiragana/Katakana toàn giác/Chữ số/Chữ cái/ー/-/Khoảng trắng.)
6. (Tùy chọn) Khi gửi với họ「山﨑」, hệ thống sẽ chuyển hướng sang dịch vụ eKYC bên ngoài. Do chưa xác nhận eKYC có hoạt động trên local hay không, nên chỉ cần chụp lại DevTools tab Network xác nhận trường `NAME` trong request `/user/ekyc_url` có chứa giá trị「山﨑」là đủ

### A-2. Cá nhân: Nghề nghiệp (職業) (l2216) — Chỉ check định dạng (không check SJIS-win)

**Các bước thực hiện**

1. `/l2200` → Tại card "Nghề nghiệp" (職業), chọn "Thay đổi" (変更) → `/l2216`
2. Chọn nghề nghiệp là nhân viên công ty (会社員) v.v., nhập các giá trị theo bảng bên dưới vào ô **Tên nơi làm việc (勤務先名)** rồi bỏ focus (Phòng ban trực thuộc / Chức vụ nhập giá trị bình thường)
3. Chụp lại hiển thị lỗi định dạng

| Nhập (Tên nơi làm việc) | Trước khi sửa | Sau khi sửa |
| --- | --- | --- |
| `山﨑商事` | 「特殊文字は入力できません。」 | Không có lỗi |
| `㐂商事` | 「特殊文字は入力できません。」 | Không có lỗi (do không có check SJIS-win) |
| `𠮷田商事` | 「特殊文字は入力できません。」 | Không có lỗi |
| `山田😀` | 「特殊文字は入力できません。」 | 「特殊文字は入力できません。」 |

### B-1. Cá nhân: Gửi (submit) Nghề nghiệp (l2216) — Lưu ký tự 4 byte (**Bắt buộc**. Xác nhận lại §0)

Chỉ thực hiện trên bản Sau khi sửa.

1. Tại màn hình của A-2, nhập tên nơi làm việc là `山﨑商事` rồi bấm gửi (submit), sau đó nhập mã PIN giao dịch → Chụp lại màn hình `/l2203` hiển thị「変更が完了しました」(Thay đổi đã hoàn tất)
2. Mở lại màn hình l2216 một lần nữa, gửi với tên nơi làm việc là `𠮷田商事`
3. Ghi lại 3 điểm sau:
   - Hiển thị trên màn hình (Alert cảnh báo lỗi / có chuyển màn hình hay không)
   - Status code và response của `/user/job` trên tab Network trong DevTools
   - Log của common-front-api (`storage/logs/`) có xuất hiện `SQLSTATE[HY000]: General error: 1366 Incorrect string value` hay không
4. Dọn dẹp (cleanup): Đổi tên nơi làm việc về lại giá trị ban đầu rồi gửi

**Dự kiến**: Bước 1 sẽ thành công. Bước 2 sẽ bị lỗi server do lỗi DB (1366).
Nếu đúng như dự kiến, hãy ghi nhận lại trên MR và Backlog, sau đó trao đổi để thống nhất phương án xử lý (tham khảo §3).

### A-3. Cá nhân: Quốc tịch (国籍) (l2213) — Chỉ check định dạng (type4)

1. `/l2200` → Tại card "Quốc tịch" (国籍), chọn "Thay đổi" (変更) → `/l2213`. Tại mục quốc tịch, chọn "Khác" (その他)
2. Nhập `山﨑国` / `㐂` / `𠮷田` / `山田😀` vào ô **Tên quốc gia (国名)** rồi bỏ focus

| | Trước khi sửa | Sau khi sửa |
| --- | --- | --- |
| 山﨑国・㐂・𠮷田 | 「漢字/ひらがな/全角カタカナ/半角英字/空白スペースを入力してください。」 | Không có lỗi |
| 山田😀 | Như trên | Như trên (lỗi) |

Vì khi bấm gửi hệ thống sẽ chuyển sang eKYC nên không thực hiện gửi.

### A-4. Cá nhân: Đăng ký người nội bộ [Quỹ] (内部者登録[基金]) (l2251) — Chỉ check định dạng (type12)

1. `/l2200` → "Đăng ký người nội bộ của quỹ do công ty chúng tôi quản lý" (当社取扱ファンド内部者登録) → `/l2251`. Chọn có người nội bộ (内部者あり)
2. Nhập `山﨑` / `㐂` / `𠮷田` / `山田😀` vào ô **Họ tên người nội bộ (内部者氏名)** rồi bỏ focus

| | Trước khi sửa | Sau khi sửa |
| --- | --- | --- |
| 山﨑・㐂・𠮷田 | 「漢字/ひらがな/全角カタカナ/全角英字/半角英字で入力してください」 | Không có lỗi |
| 山田😀 | Như trên | Như trên (lỗi) |

(Tùy chọn) Nếu gửi với `𠮷田`, dự kiến sẽ bị lỗi DB tương tự như B-1. Trường hợp muốn kiểm tra, sau khi gửi hãy đổi lại giá trị ban đầu.

### A-5. Nhóm xác minh danh tính: Thiết lập lại mã PIN giao dịch (取引暗証番号再設定) (l2500) — Có đối chiếu với họ tên đăng ký

1. Đăng nhập bằng tài khoản cá nhân → Cài đặt (設定) → "Trường hợp quên mã PIN giao dịch bấm vào đây" (取引暗証番号をお忘れの方はこちら) → `/l2500`
2. Nhập `山﨑` vào phần họ của ô **Họ và tên (toàn giác)** (お名前（全角）) rồi bỏ focus

| | Trước khi sửa | Sau khi sửa |
| --- | --- | --- |
| Họ `山﨑` | 「お名前を正しく入力してください。」 | Không có lỗi |
| Họ `山田😀` | 「お名前を正しく入力してください。」 | Như trên |

3. Sau khi sửa, dùng tài khoản có họ tên đăng ký là「崎」để nhập họ `山﨑`, nhập chính xác ngày tháng năm sinh và số điện thoại rồi gửi → Chụp lại màn hình hiển thị lỗi từ server với nội dung thông tin nhập vào không trùng khớp (thay đổi hành vi này đã được chấp nhận trong ADR 0002)
4. (Tùy chọn) Trường hợp có thể chuẩn bị tài khoản có họ tên đăng ký chứa chữ「﨑」, gửi với họ `山﨑` và chụp lại màn hình chuyển tiếp được sang `/l2501` (xác thực 2 bước)

### A-6. Pháp nhân: Tên pháp nhân (法人名) (l2221) — Check định dạng (type11) ＋ check SJIS-win

1. Đăng nhập bằng tài khoản pháp nhân → Cài đặt (設定) → Xác nhận / Thay đổi thông tin khách hàng (お客様情報の確認・変更) → `/l2900` → Card "Tên pháp nhân" (法人名) → `/l2221`
2. Nhập vào ô **Tên pháp nhân (toàn giác)** (法人名（全角）) rồi bỏ focus

| Nhập | Trước khi sửa | Sau khi sửa |
| --- | --- | --- |
| `株式会社山﨑` | 「法人名を全角で入力してください。」 | Không có lỗi |
| `株式会社㐂` | Như trên | 「㐂の入力はできません。ひらがなで入力してください。」 |
| `株式会社𠮷田` | Như trên | 「𠮷の入力はできません。ひらがなで入力してください。」 |
| `株式会社😀` | Như trên | 「法人名を全角で入力してください。」 |

### A-7. Pháp nhân: Địa chỉ trụ sở (所在地) (l2223) — Địa chỉ (type3・type7) ＋ check SJIS-win

1. `/l2900` → Card "Địa chỉ trụ sở" (所在地) → `/l2223`
2. Nhập các giá trị giống như Bước 5 của A-1 vào Quận/Huyện/Xã/Phường, Số nhà/Ngõ ngách, Tên tòa nhà để kiểm tra (kết quả kỳ vọng cũng giống như phần địa chỉ của A-1)

### B-2. Các màn hình có cùng cơ chế (Tùy chọn)

| Case | Màn hình | Cách truy cập | Ô nhập (type) | Check SJIS-win |
| --- | --- | --- | --- | --- |
| B-2a | l2228 Thông tin người đại diện (代表者情報) | `/l2900` → Họ tên người đại diện (代表者氏名) | Họ và tên người đại diện Họ/Tên (1) (代表者のお名前 姓/名) | Có |
| B-2b | l2229 Thông tin người phụ trách giao dịch (取引責任者情報) | `/l2900` → Người phụ trách giao dịch (取引責任者) | Họ và tên (1) (お名前)・Địa chỉ (3・7) (住所) | Có |
| B-2c | l2232 Người hưởng lợi thực tế (実質的支配者) (bao gồm ruleAddress) | `/l2900` → Người hưởng lợi thực tế 1 (実質的支配者1) | Họ/Tên (1) (姓/名)・Tên pháp nhân (11) (法人の名称)・Địa chỉ (3・7) (住所) | Có |
| B-2d | x1201 Mở khóa tài khoản (アカウントロック解除) | Lỗi khóa tại màn hình đăng nhập → Yêu cầu mở khóa (申請) | Họ và tên (1) (お名前) | Không (Có đối chiếu) |
| B-2e | y1200 Quên ID đăng nhập (ログインID忘れ) | Màn hình đăng nhập "Trường hợp quên Login ID bấm vào đây" (ログインIDを忘れた方はこちら) | Họ và tên (1) (お名前) | Không (Có đối chiếu) |
| B-2f | y2100 Quên mật khẩu (パスワード忘れ) | Màn hình đăng nhập "Trường hợp quên mật khẩu bấm vào đây" (パスワードを忘れた方はこちら) | Họ và tên (1) (お名前) | Không (Có đối chiếu) |
| B-2g | x2301 Mở khóa giao dịch (取引ロック解除) | Cài đặt (設定) → Mở khóa giao dịch (取引ロック解除) | Họ và tên (1) (お名前) | Không (Có đối chiếu) |
| B-2h | l2504・x1204・x2304・y1204・y2104 (Tab pháp nhân / 法人タブ) | Tab pháp nhân trên từng màn hình | Tên pháp nhân (11) (法人名)・Họ và tên người phụ trách giao dịch (1) (取引責任者のお名前) | Không (Có đối chiếu) |

Giá trị kỳ vọng: Các màn hình có check SJIS-win tương tự như A-1, các màn hình không có check SJIS-win tương tự như A-5.

l2217 (Đăng ký người nội bộ / 内部者登録) do route đã bị comment out nên không thể vào màn hình, do đó không thuộc phạm vi đối tượng.

---

## 3. Cách tổng hợp kết quả (Dán vào MR)

| Case | Trước khi sửa | Sau khi sửa | Kết quả | Ghi chú |
| --- | --- | --- | --- | --- |
| A-1 l2212 Họ「山﨑」 | Lỗi định dạng | Không có lỗi | OK / NG | |
| A-1 l2212 Họ「𠮷田」 | Lỗi định dạng | Lỗi SJIS-win (E103-1045) | | |
| B-1 l2216 Gửi「𠮷田商事」 | (Không thể gửi do lỗi định dạng) | Điền kết quả thực tế | | Status code・Log |
| … | | | | |

Trường hợp B-1 xảy ra lỗi DB đúng như dự kiến, hãy ghi kết quả vào MR và trao đổi để chọn một trong hai hướng sau:

- Tạm thời loại bỏ phạm vi 4 byte (U+20000〜U+3FFFF, flag `u`. Commit `dceef81a`) trong lần này
- Bổ sung `checkJpText` cho các màn hình không gọi check SJIS-win (ticket riêng)
