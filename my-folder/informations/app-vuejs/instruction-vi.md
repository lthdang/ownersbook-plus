# 1. Tổng quan dự án
- `app-vuejs` (được khai báo là `st_fund_app_vuejs` trong `package.json`) đóng vai trò là ứng dụng SPA phía front-end cho nền tảng giao dịch chứng khoán tài sản bảo đảm (ST).
- Ứng dụng web này cho phép nhà đầu tư trực tuyến thực hiện các nhiệm vụ hằng ngày như đăng nhập, tìm kiếm hoặc xem chi tiết sản phẩm ST, đặt lệnh mua/bán, gửi/rút tiền và cập nhật thông tin cá nhân.

# 2. Mục tiêu dự án
- Giao dịch Chứng khoán số (ST): Cho phép người dùng tìm kiếm và xem chi tiết các dự án bất động sản được số hóa ở cả thị trường sơ cấp (phát hành mới) và thứ cấp. Nó hỗ trợ đăng ký ưu tiên mua hoặc tham gia xổ số trước bán, đồng thời hướng dẫn người dùng qua quy trình đặt lệnh mua/bán nghiêm ngặt theo quy định pháp luật (bao gồm xác minh tài liệu, đánh giá phù hợp với đầu tư và kiểm tra tình trạng nội bộ).
- Quản lý tài khoản & Xác thực an toàn: Tạo điều kiện cho việc đăng nhập, đăng ký/cập nhật thông tin cá nhân và doanh nghiệp, thiết lập mã PIN giao dịch, xác thực tài khoản chứng khoán và quy trình khôi phục tài khoản.
- Quản lý dòng tiền & Tài sản: Cung cấp chức năng nạp tiền, phát lệnh rút tiền, theo dõi biến động ví/số dư và xem tổng quan tài sản.
- Tra cứu lịch sử & Báo cáo pháp lý: Cho phép nhà đầu tư tra cứu lịch sử đặt lệnh và thực hiện giao dịch, cũng như hồ sơ phân phối lợi nhuận và hoàn trả gốc; xem báo cáo điện tử; và tải xuống tài liệu để hỗ trợ quyết toán thuế.

# 3. Ngăn xếp công nghệ
Ngăn xếp công nghệ của dự án `app-vuejs` dựa trên Vue 2, bắt nguồn từ template `vue-cli` năm 2021. Các công nghệ và thư viện cụ thể được sử dụng bao gồm:
- Nền tảng Core Framework & UI:
  - Vue.js: 2.6.14 (Core UI Framework).
  - Vue Router: 3.5.3 (Routing + auth guard).
  - Vuex: 3.6.2 (Quản lý trạng thái dùng chung được sử dụng, nhưng phạm vi rất hạn chế; ứng dụng chủ yếu dựa vào `sessionStorage`).
  - Framework7 / Framework7-Vue: 6.3.7 / 5.7.14 (Lớp UI di động: xử lý hộp thoại, chỉ báo tải, popup và panel bên).

- Công cụ build & Chất lượng mã:
  - Webpack: ^3.6.0 (Sử dụng ba tệp cấu hình trong `build/`: `dev`, `dev.testing`, và `prod`.)
  - Babel: ^6 (Sử dụng preset-env và stage-2 để transpile ES2015+ và Vue JSX.)
  - ESLint: ^4.15.0 (Tuân thủ cấu hình `standard` và `plugin:vue/essential`, và tự động chạy thông qua `eslint-loader` trong mọi build.)
  - Bộ kiểm thử: Jest (^22.0.4) cho unit testing và Nightwatch + Selenium (^0.9.12) cho E2E testing.

- Truyền thông Backend:
  - axios: 0.24.0 (Một HTTP client dùng chung với timeout 80 giây giao tiếp với ba hệ thống API: Passport, Common và Fund API.)

- Xử lý dữ liệu & Thư viện chức năng:
  - Đồ thị & Xem tài liệu: `chart.js` (^2.9.4) và `vue-chartjs` (^3.5.1) để hiển thị biểu đồ giá; `vue-pdf` (^4.2.0) tích hợp trình xem `pdf.js` đi kèm để hiển thị báo cáo và tài liệu.
  - Thời gian & Tính toán: `moment` (2.29.1) / `moment-timezone`, `numeral` (^2.0.6), và `mathjs` (^10.4.0) để định dạng và tính toán số tiền và số đơn vị.
  - Bảo mật & Mã hóa: `crypto-js` (^4.2.0) / `sha256` (^0.2.0) được dùng để mã hóa mật khẩu trước khi gửi qua API; `sanitize-html` (^2.4.0) được dùng để làm sạch HTML từ máy chủ.
  - Xác thực dữ liệu: `libphonenumber-js` (^1.9.15), `email-validator` (^2.0.4).

- Giám sát & Gỡ lỗi:
  - Sentry: `@sentry/vue` & `@sentry/tracing` (^6.19.2) được dùng để giám sát và thu thập lỗi runtime.
  - vConsole: ^3.4.1 (Console hiển thị trên màn hình thiết bị di động thực trong môi trường Dev/Testing).

* Ghi chú về cơ chế CDN Externals: các thư viện cốt lõi như Vue, Vue Router, Vuex, axios, moment và Framework7 không được đóng gói vào ứng dụng. Thay vào đó, chúng được khai báo trong cấu hình `externals` của Webpack và tải trực tiếp qua CDN (jsDelivr) trong tệp `index.html`. Do đó, phiên bản thực tế đang chạy trong trình duyệt phụ thuộc vào thẻ `<script>` trong `index.html` thay vì chỉ phụ thuộc vào `package.json`.

# 4. Cấu trúc dự án
- Thư mục mã nguồn chính (`src/`):
  - `src/main.js`: Tệp quan trọng nhất của dự án. Nó đóng vai trò là entry point chịu trách nhiệm khởi tạo Sentry, đăng ký bộ các component dùng chung, chèn module tiện ích vào `Vue.prototype`, và chứa toàn bộ logic Router Auth Guard (`router.beforeEach`).
  - `src/App.vue`: Root component: Framework7 framework, animation chuyển màn hình, khởi tạo widget Zendesk và gọi API `/init`.
  - `src/router/index.js`: ~1,600 dòng, 170 route, mỗi route được lazy-load qua `require.ensure`.
  - `src/store/index.js`: Một tệp Vuex rất ngắn (48 dòng) chỉ quản lý 4 biến trạng thái: cấu hình dialog lỗi, trạng thái thông báo chưa đọc, thông tin người dùng.
  - `src/views/Resiger/`: Chứa 174 màn hình chính (views) và 8 thư mục con. Lưu ý: tên thư mục "Resiger" bắt nguồn từ lỗi chính tả cũ của từ "Register" nhưng đã được giữ lại vì toàn bộ router tham chiếu đến nó. Tên tệp `.vue` trong thư mục này tuân theo quy ước Mã màn hình (ví dụ: `g2107.vue`, `l2200.vue`).
  - `src/views/Login/index.vue` & `src/views/Read/read-pdf.vue`: Màn hình đăng nhập chính (route /) và màn hình đọc PDF.
  - `src/components/`: Chứa 51 component dùng chung, trong đó 45 component được đăng ký tự động như global component qua `src/components/index.js`.
  - `src/utils/`: Chứa 9 module xử lý logic dùng chung, như HTTP Client (`httpRequest.js`), hàm tiện ích chung (`common.js`), master data (`common-data.js`) và danh sách câu hỏi liên quan đến đầu tư (`contentItem.js`).
  - `src/assets/`: `assets/css/`: Hệ thống CSS toàn cục (không dùng `<style scoped>`). Thiết kế cốt lõi dựa trên `pc.css` (~70KB) và `sp.css` (~77KB) cho PC và mobile, bổ sung hai tệp override: `expand_pc.css` và `expand_sp.css`. `assets/img` và icons: Chứa 152 file SVG và 19 file hình raster..

- Thư mục cấu hình & build:
 - `build/`: Chứa các tệp cấu hình Webpack 3 cho ba môi trường: `dev`, `dev.testing`, và `prod`.
 - `config/`: Định nghĩa các biến môi trường tương ứng (`prod.env.js`, `dev.env.js`, `dev.testing.env.js`, `test.env.js`) và cấu hình dev-server.
 - `static/pdfjs/`: Trình xem PDF `pdf.js` đi kèm sẽ được sao chép nguyên vẹn vào thư mục `dist/` trong quá trình build. 
 - `test/`: Chứa các tệp mẫu cho unit tests (Jest) và E2E tests (Nightwatch).
 - `.gitlab-ci.yml`: Tệp định nghĩa pipeline CI/CD (bao gồm cấu hình cụ thể cho staging3 và staging4).
 - `.env`: Tệp chứa biến môi trường cho máy local (không commit lên Git; dựa trên template `.env.dev` hoặc `.env.testing`).

# 5. Kiến trúc
- Kiến trúc Điều hướng & Phân quyền Router:
  - Lazy Loading / Code Splitting: Hệ thống gồm 170 route được định nghĩa trong `src/router/index.js`. Mỗi màn hình được lazy-load bằng `require.ensure`, chia code thành các JS chunk độc lập theo mã màn hình cụ thể.
  - Auth Guard tập trung (`router.beforeEach`): Nằm trong `src/main.js`, đây là cơ chế trung tâm để kiểm tra trạng thái đăng nhập (qua `meta.requireAuth`) và xử lý phân quyền người dùng (Cá nhân vs. Doanh nghiệp). Nó cũng buộc chuyển hướng bắt buộc cho tài khoản chưa thiết lập PIN (m2100) hoặc chưa xác minh số tài khoản chứng khoán (m2102).
  - Cơ chế Force Reload (Tăng cường bảo mật tài chính): Đối với các màn hình giao dịch liên quan đến chuyển khoản, rút tiền hoặc mua/bán (ví dụ: g2107, g2200, j2200), Router Guard dùng `window.location.href` để thực hiện full page reload (Hard Reload). Điều này xóa hoàn toàn state ở client, đảm bảo an toàn tuyệt đối cho giao dịch tài chính.

- Quản lý trạng thái và bộ nhớ trình duyệt (State Management):
  - Vuex Store (`src/store/index.js`) — Được sử dụng rất hạn chế. Mặc dù dự án tích hợp Vuex (phiên bản 3.6.2), tệp `src/store/index.js` lại chỉ khoảng 48 dòng và lưu trữ chỉ bốn biến trạng thái chính:
    - `apiErrorMsgConfig`: Cấu hình hiển thị dialog thông báo lỗi chung (được `<my-dialog>` trong `App.vue` lắng nghe và cập nhật thông qua `commonJs.showAlert()`).
    - `noticeRead`: Cờ cho biết có thông báo chưa đọc hay không.
    - `showInviteMenu`: Cờ hiển thị/ẩn menu mời bạn bè.
    - `userInfo`: Thông tin chi tiết người dùng được lấy từ API `/user/info`.
  - Một đặc điểm nổi bật của các mutation Vuex này là mutation cho `noticeRead` và `showInviteMenu` không nhận giá trị từ payload theo cách chuẩn của Vuex; thay vào đó, chúng được thiết kế để truy xuất giá trị từ `sessionStorage` trước khi gán vào state.
  - `sessionStorage` — Nền tảng cho việc truyền dữ liệu giữa các màn hình:
    - Đặc điểm: Dữ liệu tự động mất khi tab đóng, tách riêng session cho từng tab độc lập. Nếu API trả về lỗi 401 Unauthorized, toàn bộ `sessionStorage` sẽ bị xóa.
  - Các điểm chính:
    - `ACCESS_TOKEN`: Token xác thực (sự hiện diện của khóa này đóng vai trò là cờ kiểm tra đăng nhập).
    - `USER`: Chứa JSON thông tin người dùng (`PERSON_CORP_TYPE` [cá nhân/doanh nghiệp], `IS_SECRET_CONFIGURED`, `IS_ACCOUNT_NUMBER_VERIFIED`) để sử dụng bởi Auth Guard.
    - `INIT_INFO`: Dữ liệu khởi tạo từ API `/init` (chứa số lượng quỹ `FUND_COUNT`, v.v.).
    - `secretToken`: Token xác nhận xác thực của mã PIN giao dịch.
    - `BUY_FUND_CD`, `FUND_NM`, `ORDERINFO`: Mã quỹ, tên quỹ và thông tin đơn hàng được truyền giữa các màn hình trong luồng đặt lệnh.
    - `prevPageRoute`: Lưu trữ route của màn hình trước đó để hỗ trợ logic phân nhánh.
    - `PAGE_AUTO_RELOAD`: Cờ cho biết một chuyển màn hình do hard reload.
  - `localStorage` — Bộ nhớ thiết bị lâu dài; `localStorage` được sử dụng cho thông tin cần được giữ lại lâu dài:
    - `uuid`: Một định danh thiết bị luôn được đưa vào mọi request API qua header HTTP `X-DU`. Khi người dùng đăng xuất, hệ thống thực hiện `localStorage.clear()` nhưng luôn khôi phục giá trị `uuid` này.
    - `USERNAME`: Lưu trữ ID đăng nhập đã nhớ.
    - `stoData`: Dữ liệu lưu trữ bền vững cho các màn hình cụ thể.
  - Tác động của cơ chế Force Reload (Hard Reload) đối với state:
    - Khi điều hướng đến các màn hình liên quan đến giao dịch tài chính (như mua/bán chứng khoán hoặc nạp/rút tiền—ví dụ g2107, g2200, j2200), Router Guard dùng `window.location.href` để thực hiện full browser reload (Hard Reload).
    -Cơ chế này đặt lại toàn bộ state Vuex, chỉ giữ lại dữ liệu được lưu trong `sessionStorage`. Đây là thiết kế chủ ý để tránh mang theo state client-side cũ, không phải lỗi hệ thống.

- Kiến trúc kết nối API Backend:
  - Ứng dụng giao tiếp với ba hệ thống backend độc lập qua các URL được định nghĩa trong biến môi trường:
    - `passportApiUrl`: Xử lý đăng nhập và phát hành token.
    - `commonApiUrl`: Xử lý thông tin người dùng, cài đặt, nạp/rút tiền và `/init`.
    - `fundApiUrl`: Xử lý dữ liệu đăng ký/phát hành, đặt lệnh và lịch sử giao dịch.

  - Shared HTTP Client (`src/utils/httpRequest.js`): Dự án khởi tạo một axios instance dùng chung trong `src/utils/httpRequest.js` và chèn nó dưới dạng `Vue.prototype.axios`, cho phép bất kỳ component nào truy cập qua `this.axios` với timeout cố định là 80 giây. Request Interceptor (Tự động chèn header): Mỗi request đi ra đều tự động bao gồm các header cần thiết:
    - `X-CI`: Company identifier (mã định danh công ty, suy ra từ biến môi trường `XCompanyId`).
    - `X-DU`: Device identifier (lấy từ UUID trong `localStorage`).
    - `X-SV`, `X-PF`, `X-UA`, `X-MD`, `X-OV`: Thông tin về phiên bản ứng dụng, nền tảng (iOS / Android / Web), User-Agent, model thiết bị và phiên bản OS.
    - `Authorization`: Chuỗi token xác thực `ACCESS_TOKEN` lấy trực tiếp từ `sessionStorage`.
    - `X-CT`: Dành riêng cho các API giao dịch tài chính (mua/bán, nạp/rút tiền—nằm trong mảng `transactionApi` trong `httpRequest.js`); nó tự động lấy giá trị cookie `ct` từ Nginx (bỏ qua trong môi trường dev VueJS khi `VueJsJSEnv === 'dev'`).

  - Định dạng phản hồi & Xử lý lỗi: Tất cả API hệ thống trả về cấu trúc JSON thống nhất:
    ```json
    {
    "STATUS": "OK" | "NG" | "MAINTENANCE",
    "DATA": { ... },
    "ERROR": { "CODE": "E100-0011", "MESSAGE": "..." }
    }
    ```
   * (Lưu ý: Dù HTTP status là 200 OK, developers vẫn phải kiểm tra các trường `STATUS` và `ERROR` trong body.)

  - Response Interceptor (Xử lý lỗi tự động): Response Interceptor tự động chặn và xử lý các trường hợp đặc biệt, loại bỏ nhu cầu viết lại logic cho từng màn hình:
    - Maintenance (`STATUS === "MAINTENANCE"`): Lưu URL bảo trì và điều hướng ứng dụng tới màn hình bảo trì `z1100`.
    - Response không phải JSON / Thiếu trường `STATUS`: Tự động chuyển hướng đến màn hình lỗi hệ thống `z1200`.
    - Tài khoản/thiết bị chưa được xác minh (Mã lỗi E100-0011, E100-0013, `no_account_number_verified_error`, `xdu_credible_error`): Hiển thị cảnh báo cho người dùng và chuyển hướng về trang chủ.
    - Lỗi HTTP 401 Unauthorized: Xóa toàn bộ dữ liệu trong `sessionStorage` và `localStorage` (giữ lại UUID) và chuyển hướng tới trang đăng nhập (/).
    - Lỗi HTTP 500 hoặc Mất kết nối: Chuyển hướng tự động đến màn hình lỗi hệ thống `z1200`.

  - Xử lý dữ liệu & Bảo mật:
    - Mã hóa mật khẩu & thông tin nhạy cảm: Đăng nhập và mật khẩu giao dịch không bao giờ được truyền ở dạng plaintext. Hàm `getEncryptionPassword()` trong `utils/common-api.js` tự động gọi API `/common/encryption_key` để lấy public key từ server và sử dụng `crypto-js` để mã hóa mật khẩu—cùng với ID phiên bản key—trước khi gửi qua API.
    - HTML Sanitization (XSS Sanitization): Khi hiển thị dữ liệu HTML trả về từ server qua directive `v-html`, bạn bắt buộc phải truyền dữ liệu qua `this.$sanitize(data)` (dùng thư viện `sanitize-html`) để ngăn XSS.
    - Định dạng tiền tệ & ngày: Dự án đã inject sẵn `this.$Moment` (Moment.js) cho xử lý ngày và cung cấp bộ hàm tiện ích trong `this.commonJs` để chuyển đổi giữa ký tự full-width và half-width, định dạng tiền tệ theo dấu phẩy và tính toán số lượng đơn vị.

  - Shared API Module (`src/utils/common-api.js`): Thay vì viết các API call trực tiếp trong từng component, các thao tác API được sử dụng rộng rãi trên nhiều màn hình được tập hợp trong `src/utils/common-api.js`:
    - `getInit()`: Lấy thông tin cấu hình ban đầu từ `/init`.
    - `getUserInfo()`: Gọi API `/user/info` để lấy thông tin mới nhất của nhà đầu tư và tự động cập nhật Vuex store.
    - `getEncryptionPassword()`: Thực hiện quy trình lấy key và mã hóa mật khẩu.
    - `checkJpText()`: Xác thực chuỗi ký tự (ngăn nhập Kanji vào các trường yêu cầu Katakana).
    - `getFundIcon()`: Lấy icon hiển thị cho từng mã quỹ/phát hành.

- Đăng ký component tự động & chèn tiện ích (Framework Infrastructure)
  - Global Components: 45/51 shared components nằm trong `src/components/` được đăng ký tự động dưới dạng global components trong `src/main.js`.
  - Prototype Injection: Các module tiện ích cốt lõi được chèn trực tiếp vào `Vue.prototype` trong `src/main.js` (ví dụ: `this.commonJs`, `this.commonData`, `this.contentItem`, `this.axios`, `this.$sanitize`, `this.$Moment`), cho phép mọi component truy cập trực tiếp mà không cần import lại.
  - Page Skeleton: Hầu hết các màn hình tuân theo cấu trúc HTML chuẩn hóa với hai component điều hướng riêng biệt: một cho PC (`<pcnavi>`) và một cho mobile (`<spnavi>`).

- Quản lý kiểu dáng (Global Styling Architecture):
  - CSS được quản lý qua các stylesheet toàn cục tập trung (không dùng `<style scoped></style>`).
  - Giao diện dựa vào hai tệp chính—`pc.css` (~70KB) cho desktop và `sp.css` (~77KB) cho mobile (với breakpoint 768px)—kết hợp với `expand_pc.css` và `expand_sp.css` để ghi đè hoặc bổ sung kiểu giao diện.

# 6. Quy ước coding
- Các quy tắc bắt buộc: Được định nghĩa trong `.editorconfig` và `.eslintrc.js`, tự động được kiểm tra qua `eslint-loader` trong mọi build.
  - Định dạng cơ bản: thụt lề 2 khoảng trắng, line endings LF, mã hóa UTF-8, và tự động gỡ bỏ khoảng trắng thừa ở cuối dòng.
  - Chuỗi: Chỉ dùng nháy đơn `'...'` (quotes: ['error', 'single']).
  - Bộ quy tắc lint: Tuân thủ `standard` + `plugin:vue/essential` + `eslint:recommended`.
  - Biến & Debugging: Khai báo biến không dùng được coi là lỗi (ngoại trừ tham số hàm). `console` và `debugger` được xem là lỗi trong production build. Tắt kiểm tra cho khoảng trắng full-width để cho phép comment bằng tiếng Nhật.
- Quy ước đặt tên màn hình và tệp:
  - Mã màn hình: Được đặt theo công thức [Business domain code letter] + [4 chữ số] (ví dụ: g2107, l2200).
  - Tên file & Route:
    - View Components nằm trong thư mục `src/views/Resiger/`. Tên file viết thường, tương ứng với mã màn hình (ví dụ: `g2107.vue`, `l2300.vue`).
    - Cả route path và route name đều dùng mã màn hình: `/g2107`.
    - Tên screen code này được sử dụng nhất quán cả trong ticket (Backlog/Jira) và tài liệu thiết kế.
  - Shared Components:
    - Tên file nằm trong `src/components/` và bắt đầu bằng chữ 'c' theo kiểu camelCase (ví dụ: `cTextInput.vue`, `cSinglePopup.vue`).
    - Đăng ký global tự động viết hoa chữ cái đầu tiên (CTextInput), cho phép gọi trong template bằng `<cTextInput>` hoặc `<c-text-input>`.
- Quy ước thực tiễn trong codebase:
  - Điều hướng màn hình: Thường được gọi qua helper `this.commonJs.toNext('/l2100')` thay vì dùng `this.$router.push` trực tiếp.
  - API calls: Phải dùng shared `this.axios` instance (được inject vào Vue prototype); không import axios trực tiếp trong từng tệp `.vue`.
  - Quản lý ngôn ngữ/văn bản: Thông báo người dùng nên ưu tiên được định nghĩa trong dictionary `src/utils/custom_language_jp.js` thay vì hard-code trong template.
  - Quy ước comment: Codebase chứa hỗn hợp tiếng Nhật, tiếng Trung và tiếng Anh (do nhiều vendor tham gia). Comment mới phải viết bằng tiếng Nhật; tuy nhiên, các comment tiếng Trung hiện có không nên bị xóa hoặc dịch lại tùy tiện để bảo toàn ý định của tác giả gốc.
  - Quy tắc tương ứng Cá nhân/Doanh nghiệp: Nhiều luồng nghiệp vụ sử dụng màn hình riêng cho khách hàng cá nhân và doanh nghiệp (ví dụ: l2200 cho cá nhân vs. l2900 cho doanh nghiệp). Khi chỉnh sửa, developers bắt buộc phải kiểm tra và cập nhật các màn hình tương ứng.
- Quy ước Branch và Commit Message:
  - Tên Branch: Theo định dạng `backlog/<TICKET_ID>` (ví dụ: `backlog/HASHST_DEV-4400`).
  - Commit Message: Đặt Ticket ID và tiêu đề ticket ở đầu, sau đó là mô tả chi tiết bằng tiếng Nhật.

# 7. Hướng dẫn Component
- Phân loại và quy ước đặt tên Component.
  - View Components (184 files): Nằm trong `src/views/Resiger/`, đại diện cho từng màn hình riêng lẻ. Tên file được viết thường dựa trên mã màn hình (ví dụ: `g2107.vue`, `l2300.vue`).
  - Shared Components (51 files): Nằm trong `src/components/`, chứa các UI component dùng chung. Tên file bắt đầu bằng chữ 'c' và theo kiểu camelCase (ví dụ: `cTextInput.vue`, `cSinglePopup.vue`).
- Cơ chế đăng ký Component (Registration Rules):
  - Đăng ký global tự động: Tệp `src/components/index.js` export 45 shared components. Tệp `src/main.js` tự động lặp qua list này, viết hoa chữ cái đầu tiên (ví dụ: `cTextInput` $\rightarrow$ `CTextInput`) và đăng ký dưới dạng global component. Trong template của bất kỳ màn hình nào, bạn có thể gọi trực tiếp như `<cTextInput>` hoặc `<c-text-input>` mà không cần import lại.
  - Yêu cầu bắt buộc khi tạo shared component mới: Nếu bạn tạo một shared component mới trong `src/components/`, bạn phải khai báo import và export trong `src/components/index.js`; nếu không, component sẽ không được đăng ký global.
  - Local Components: Có sáu component trong `src/components/` không được đăng ký global (ví dụ: `cMulChioce.vue`, `rollerNumber.vue`). Để sử dụng chúng, bạn phải import trực tiếp vào tệp `.vue` tương ứng.
- Quy ước Page Skeleton chuẩn:
```html
<div id="page">
  <div class="pc_bg">
    <div class="base_container">
      <spnavi></spnavi>         <!-- Mobile Navigation -->
      <pcnavi></pcnavi>         <!-- PC Navigation-->
      <div class="right_area">
        <div class="site_path onlypc"> ... </div>   <!-- Breadcrumb (Only displays on PC) -->
        <subpageheader></subpageheader>
        <div id="main">
          <div class="wrapper">
            <!-- Screen-specific content here -->
          </div>
        </div>
      </div>
    </div>
    <subpagefooter></subpagefooter>
  </div>
</div>
```
- Lưu ý về giao diện PC & Mobile: `<pcnavi>` và `<spnavi>` là hai component riêng biệt, bật/tắt dựa trên media query CSS (breakpoint 768px). Khi sửa menu hoặc navigation, developers phải cập nhật cả hai component. Dự án cũng có các biến thể ẩn menu như `pcnaviMnone` và `spnaviMnone`.

- Các Shared Components cốt lõi (Key Input Components):
  - Nhập dữ liệu cơ bản: `cTextInput`, `cTextInputNotSpace`, `cNumInput`, `cDecimalInput` (có validate tích hợp).
  - Bộ chọn (Select / Radio): `cSelect`, `cDropDown`, `cRadio` (hoặc `cDRadio` từ `Input/radio/cRadio.vue`), `cCheckBox`.
  - Popup / Modal: `cSinglePopup`, `cSearchPopup`, `cPostalPopup` (tìm kiếm địa chỉ theo mã bưu điện), `cSingleZonePopup`.
  - Picker Mobile: `scrollerPicker`, `inputPicker`, `datePicker` (roller picker kiểu mobile).
  - Khối input kết hợp: `cAddress`, `cNameBirthdaySex` và bốn component khai báo thông tin insider (`cInsiderFlg`, `cInsiderInfo`, `cInsiderAdd`, `cInsiderSeparation`).
  - Thông báo & Lỗi: `myDialog` (shared notification dialog), `cErrors`, `cTips`, `Cnotice`.
  - Nút: `Button`, `cRound Button`, `cNextOrBack Button` (dùng class `disable` để ẩn hoặc vô hiệu hóa).

- Hướng dẫn kỹ thuật cần lưu ý khi viết component:
  - Không dùng Scoped CSS: Dự án gần như không dùng `<style scoped></>`. Tất cả CSS đều nằm trong bốn stylesheet toàn cục (`pc.css`, `sp.css`, `expand_pc.css`, `expand_sp.css`). Tránh tạo tên class trùng lặp trong component; nếu cần chỉnh sửa UI, hãy thêm một class mới vào `expand_pc.css` hoặc `expand_sp.css`.
  - Tận dụng tiện ích đã inject: Không import lại module tiện ích. Gọi trực tiếp qua prototype: `this.axios`, `this.commonJs`, `this.commonData`, `this.contentItem`.
  - An toàn dữ liệu với `v-html`: Nếu component render một chuỗi HTML từ server bằng directive `v-html`, bắt buộc phải bọc dữ liệu bằng `this.$sanitize()` để ngăn XSS (`v-html="$sanitize(content)"`).
  - Tắt hiệu ứng loading (Framework7 Preloader): Trạng thái hiển thị loading do Framework7 quản lý (`this.$f7.preloader.show()` và `this.$f7.preloader.hide()`). Luôn đảm bảo `.hide()` được gọi trong các block xử lý lỗi API để tránh màn hình bị đóng băng.

# 8. Kiểm thử
- Thực hiện cực kỳ cẩn thận với các màn hình giao dịch tài chính: Khi chỉnh sửa màn hình dòng tiền (như chức năng mua/bán chứng khoán hoặc nạp/rút quỹ như g2107, g2200, j2200, v.v.), developers bắt buộc phải thực hiện manual testing kỹ lưỡng.
- Kiểm thử trên cả hai loại tài khoản (Cá nhân & Doanh nghiệp) là bắt buộc: Vì nhiều quy trình có màn hình riêng cho khách hàng Cá nhân (`PERSON_CORP_TYPE = 1`) và Doanh nghiệp (`PERSON_CORP_TYPE = 2`) (ví dụ: l2200 vs. l2900), rất dễ xảy ra trường hợp test một phía nhưng bỏ sót phía còn lại.
- Kiểm thử trên cả hai giao diện (Desktop PC và Mobile Smartphone) là bắt buộc: PC và mobile sử dụng các navigation component hoàn toàn khác nhau (`pcnavi` / `spnavi`) và CSS file khác nhau (`pc.css` / `sp.css`); do đó, việc chỉnh sửa cho một màn hình rất dễ làm hỏng layout trên thiết bị còn lại.

# 9. Thay đổi bị cấm / nguy hiểm
- Bảo mật dữ liệu & ngăn rò rỉ dữ liệu nghiêm ngặt
  - Không dùng `v-html` trực tiếp: Bạn phải bọc dữ liệu bằng `this.$sanitize()` để ngăn XSS.
  - Không gửi mật khẩu/PIN ở dạng plain text: Bạn phải dùng helper `getEncryptionPassword()` trong `common-api.js` để lấy key mã hóa từ server trước khi truyền.

- Thay đổi source code & identifier nguy hiểm
  - Không đổi tên tuỳ tiện các identifier hiện có (`HASHST`, `hashdash`, `hhd-`): Dù các tên này bắt nguồn từ thương hiệu cũ (Hash DasH / Hash ST), hiện chúng là biến môi trường và tên dịch vụ thực tế trong AWS ECS và CI/CD workflows. Việc đổi tên có thể gây gián đoạn hệ thống hoặc phá hỏng deployment pipelines.
  - Không sửa tên thư mục bị đánh vần sai `src/views/Resiger/`: Dù tên này là lỗi chính tả của "Register", tất cả 170 route trong `src/router/index.js` đều tham chiếu trực tiếp tới nó.
  - Không đổi tên tuỳ tiện các class CSS hiện có trong `pc.css` hoặc `sp.css`. Vì dự án hiếm khi dùng `<style scoped></style>`, CSS có phạm vi toàn cục; đổi tên class hiện có dễ làm hỏng layout trên các màn hình khác. Khi cần sửa đổi, hãy thêm class mới vào `expand_pc.css` hoặc `expand_sp.css` thay vì thay đổi hiện có.
  - Không nâng cấp library một cách tùy tiện lên Vue 3: Dự án đang chạy trên nền Vue 2 cũ (đã hết hỗ trợ) với Webpack 3 và Babel 6. Tính tương thích với Vue 2 phải được kiểm tra kỹ trước khi thêm bất kỳ dependency nào.

- Các quy định liên quan đến Git và CI/CD:
  - Không commit tệp `.env` lên Git: Tệp `.env` chứa chi tiết cấu hình local và được liệt kê trong `.gitignore`. Không được push tệp này lên repository dưới bất kỳ hình thức nào.
  - Push trực tiếp vào nhánh `master` bị cấm: Tất cả thay đổi dành cho `master` phải được nộp qua Merge Request (MR) để review.
  - Tránh push "im lặng" lên các nhánh testing: Pushing code lên nhánh như `testing`, `testing_ph2` hoặc `testing3` sẽ kích hoạt pipeline CI/CD và triển khai ngay lên shared test server, có thể làm gián đoạn hoạt động testing của các thành viên khác.

# 10. Tài liệu tham khảo quan trọng
- Tình trạng tài liệu tham khảo trong Repository:
  - Các file interface reference (sample1 ~ sample10): Nằm trong `src/views/Resiger/`, đây là các template màn hình đã dựng sẵn. Dù không chạy trong môi trường production, chúng vẫn đóng vai trò là tài liệu tham khảo về cấu trúc khi tạo màn hình mới.
- Glossary:
  - Tài chính: ST (Digital Securities), Primary/Secondary, Priority application/Lottery, Order matching, Number of trading units, Suitability questionnaire, eKYC (Electronic Know Your Customer), Transaction PIN.
  - Kỹ thuật: SPA, Routing/Route Guard, Code Splitting/Lazy Loading, Interceptor, `sessionStorage`/`localStorage`, ECS/ECR/CodeDeploy, Blue/Green Deployment.
