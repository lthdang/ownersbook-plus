# Quy trình tạo Ledger OwnersBook+ và các API liên quan (Hướng dẫn nhập môn kiểu ELI5)

## Mục lục
1. ["Ledger" là gì? (Ví dụ trong đời thực)](#1-what-is-a-ledger-the-real-world-analogy)
2. [Bức tranh lớn: Hai loại Ledger (Loại A và Loại B)](#2-the-big-picture-two-types-of-ledgers-type-a-vs-type-b)
3. [Loại A — Quy trình tạo Ledger theo quy định (Từng bước)](#3-type-a--the-statutory-ledger-creation-flow-step-by-step)
   - [Bước 1: Khởi tạo giao dịch và ghi nhận Escrow (`common-front-api`)](#step-1-transaction-initiation--escrow-recording-common-front-api)
   - [Bước 2: Lên lịch Batch và kích hoạt chạy (`st-cli` / Airflow)](#step-2-batch-scheduling--invocation-trigger-st-cli--airflow)
   - [Bước 3: Kiểm tra trước khi chạy và kiểm tra lịch (`st-cli`)](#step-3-pre-execution-validation--calendar-check-st-cli)
   - [Bước 4: Lấy dữ liệu và xử lý theo quy định (`st-cli`)](#step-4-data-retrieval--statutory-processing-st-cli)
   - [Bước 5: Tạo và định dạng PDF/Report (`st-cli`)](#step-5-pdfreport-generation--formatting-st-cli)
   - [Bước 6: Lưu trữ trên S3, publish thư mục và dọn dẹp (`st-cli`)](#step-6-s3-archival-directory-publishing--cleanup-st-cli)
   - [Bước 7: Truy cập Backoffice và phân phối điện tử (`st-admin` / `common-front-api`)](#step-7-backoffice-access--electronic-delivery-st-admin--common-front-api)
4. [Làm quen với ba Ledger theo quy định: RT003, RT004 và RT005](#4-meet-the-three-statutory-ledgers-rt003-rt004-and-rt005)
5. [Loại B — Distributed Ledger (GoQuorum On-Chain)](#5-type-b--the-distributed-ledger-on-chain-goquorum)
6. [Bảng tham chiếu API đầy đủ](#6-comprehensive-api-reference-table)
7. [Điểm cần lưu ý và các lỗi thường gặp](#7-points-to-note--common-gotchas)

---

## 1. "Ledger" là gì? (Ví dụ trong đời thực)

Hãy tưởng tượng bạn điều hành một hiệu sách trong trường, có một câu lạc bộ tiết kiệm cho học sinh.

Mỗi khi một học sinh gửi tiền tiêu vặt, mua sách giáo khoa hoặc rút phần tiền ăn trưa còn dư, bạn ghi lại vào một quyển sổ.

Quyển sổ ghi chép đặc biệt đó được gọi là **ledger**.

Ledger chỉ đơn giản là nơi theo dõi ai sở hữu bao nhiêu tiền và những món hàng nào đã được chuyển giao theo thời gian.

Trong hệ thống của chúng ta, các quy định tài chính yêu cầu phải in ra những bản ghi chính thức, không thể sửa đổi để chứng minh công ty đã xử lý chính xác từng đồng yên.

---

## 2. Bức tranh lớn: Hai loại Ledger (Loại A và Loại B)

Khi developer nói về một "ledger" trong OwnersBook+, họ có thể đang nói đến hai thứ hoàn toàn khác nhau và không được nhầm lẫn:

### 1. Loại A — Statutory Ledger (Chủ đề chính)
- **Là gì**: Các báo cáo tài chính chính thức dạng PDF và CSV mà cơ quan quản lý yêu cầu.
- **Ví dụ**: Một sao kê ngân hàng hàng tháng được accountant in ra và đóng dấu.
- **Cách chạy**: Một robot tự động chạy nền, gọi là **batch**, đọc database vào ban đêm và tạo các file PDF.
- **Codebase chính**: Repository `st-cli`.

### 2. Loại B — Distributed Ledger (Thành phần phụ)
- **Là gì**: Các token quyền sở hữu kỹ thuật số nằm trên một mạng blockchain riêng gọi là **GoQuorum**.
- **Ví dụ**: Một database kỹ thuật số dùng chung, nơi mọi người đều có bản sao có thể kiểm tra được về ai sở hữu từng security token.
- **Cách chạy**: Smart contract xử lý việc chuyển token theo thời gian thực, còn chúng ta xem chúng bằng một web screen.
- **Codebase chính**: Repository `st-bc-explorer`.

> **Quy tắc quan trọng**: Loại A và Loại B hoàn toàn tách biệt. Loại A không nói chuyện trực tiếp với blockchain, còn Loại B không tạo các tài liệu PDF của chính phủ.

---

## 3. Loại A — Quy trình tạo Ledger theo quy định (Từng bước)

Quy trình tạo statutory ledger gồm **7 bước liên tiếp** từ đầu đến cuối.

### Bước 1: Khởi tạo giao dịch và ghi nhận Escrow (`common-front-api`)
- Người dùng mở web app hoặc mobile app để nạp tiền, mua phần đầu tư hoặc rút tiền mặt.
- Một API **BFF (Backend-For-Frontend)** trong `common-front-api` nhận các request này.
- Khi investor nạp tiền, họ gửi tiền vào một tài khoản giữ đặc biệt gọi là **escrow account**.
- Escrow account là một tài khoản ngân hàng được bảo vệ, nơi tiền của khách hàng được giữ an toàn và tách khỏi tiền vận hành của công ty.
- Controller `WithdrawController.php` quản lý các request rút tiền và kiểm tra số dư.
- Controller `UserPaymentController.php` nhận thông báo chuyển khoản ngân hàng từ NTT Data.
- Mọi biến động tài chính được lưu ngay vào các database table có tên `ECO_TRADE_HIST` và `PST_ALL_TRADE_HIST`.

### Bước 2: Lên lịch Batch và kích hoạt chạy (`st-cli` / Airflow)
- Chúng ta không tạo các PDF report nặng trong lúc người dùng đang click trên website.
- Thay vào đó, chúng ta chạy một **batch job**, tức là một computer script tự động chạy ở background mà không cần người can thiệp.
- Một job scheduler tên **Apache Airflow** kích hoạt batch vào ban đêm bên trong một Docker container.
- Khi test ở local, engineer kích hoạt các job bằng script tên `bulk_exec_batch.sh`.
- Các batch file chính trong `st-cli/Report/run/` là `pdf003.php`, `pdf004.php` và `pdf005.php`.
- Mỗi script nhận các command-line parameter cho biết chạy cho từng customer hay cho fund operator, và chạy theo ngày hay theo tháng.

### Bước 3: Kiểm tra trước khi chạy và kiểm tra lịch (`st-cli`)
- Trước khi thực hiện các phép tính nặng, script chạy các safety check.
- Script kiểm tra ngày mục tiêu và ghi message `START` vào execution history table.
- Script kiểm tra business calendar table tên `PST_BUSINESS_CALENDAR` để xem ngân hàng Nhật Bản có mở cửa hay không.
- Nếu hôm nay là cuối tuần hoặc ngày nghỉ lễ, script ghi `OK(CLOSING)` vào log rồi dừng yên lặng với exit code 0.
- Với ledger hàng tháng, script kiểm tra hôm nay có thật sự là ngày cuối cùng của tháng hay không.
- Script cũng load các system configuration setting từ `ESYS_CONFIG` để biết số trang tối đa được phép trong mỗi PDF.

### Bước 4: Lấy dữ liệu và xử lý theo quy định (`st-cli`)
- Khi các bước kiểm tra đạt, batch query database để lấy tất cả giao dịch đã được xác nhận.
- Với RT003, `Pdf003Model.php` đọc số dư tiền mặt và các biến động giao dịch cho tối đa 1.000 customer mỗi lần.
- Với RT004, `Pdf004Model.php` lấy số dư security token unit và lịch sử contract.
- Với RT005, `Pdf005Model.php` tổng hợp các giao dịch cấp fund, khoản trả dividend và khoản hoàn vốn cho investor.
- Script tính số dư rollover đầu kỳ, cộng tiền vào, trừ các fee phát sinh và kiểm tra số dư cuối kỳ.

### Bước 5: Tạo và định dạng PDF/Report (`st-cli`)
- Hệ thống phải biến các con số thô thành những tài liệu kế toán gọn gàng, dễ đọc.
- Hệ thống ghi các chỉ dẫn layout chính xác vào một text file tạm, bao gồm ranh giới trang và các coordinate box.
- Hệ thống sử dụng các visual template được thiết kế sẵn tên là `RT003.pdf.tpl`, `RT004.pdf.tpl` và `RT005.pdf.tpl`.
- Hệ thống chạy một chương trình PDF compilation engine nội bộ tên `PDF_MAKER_EXEC` (FastPDFGen).
- Nếu tài liệu có quá nhiều customer hoặc vượt giới hạn số trang, script sẽ tách tài liệu thành nhiều phần (`_001.pdf`, `_002.pdf`).

### Bước 6: Lưu trữ trên S3, publish thư mục và dọn dẹp (`st-cli`)
- Khi PDF được compile thành công trên disk, file đó phải được lưu lại lâu dài.
- Helper function `upload2S3ForAdmin` trong `pdf.php` upload file lên Amazon S3 cloud storage.
- Script cũng chuyển file vào các local folder có tổ chức tên `Report/manage/` và `Report/customer/`.
- Batch xóa các text file tạm và scratch folder để server luôn sạch sẽ.
- Cuối cùng, batch cập nhật database log để đánh dấu job là `OK` hoặc `NG` (No Good / Error).

### Bước 7: Truy cập Backoffice và phân phối điện tử (`st-admin` / `common-front-api`)
- Backoffice administrator của công ty đăng nhập vào management portal được host bởi `st-admin`.
- Các rule của Nginx web server cho phép administrator xem và download report thông qua `/report_manage/` và `/report_customer/`.
- Đồng thời, investor là end-user có thể xem các disclosure chính thức của chính mình thông qua `ElectronicController.php` trong `common-front-api`.
- Khi investor mở và xác nhận đã xem report, hệ thống đánh dấu tài liệu là đã đọc và đã xác nhận.

---

## 4. Làm quen với ba Ledger theo quy định: RT003, RT004 và RT005

| Mã Ledger | Tên thường gọi | Ví dụ trong đời thực | Theo dõi ai và điều gì | Script và template chính |
| :--- | :--- | :--- | :--- | :--- |
| **RT003** | Cash Customer Ledger (金銭顧客勘定元帳) | Một sổ ngân hàng cá nhân cho biết tiền vào và tiền ra. | Theo dõi số dư tiền yên, tiền nạp, tiền rút và các lần chuyển tiền escrow của từng customer và operator. | `pdf003.php`<br/>`RT003.pdf.tpl` |
| **RT004** | Merchandise Customer Ledger (商品顧客勘定元帳) | Một bản sao kê danh mục đầu tư cho biết bạn đang sở hữu bao nhiêu cổ phần. | Theo dõi số security token unit (口数) mà mỗi customer mua hoặc bán trong từng investment fund cụ thể. | `pdf004.php`<br/>`RT004.pdf.tpl` |
| **RT005** | Trading Product Ledger (トレーディング商品勘定元帳) | Một sổ kho hàng dành cho chủ cửa hàng. | Theo dõi tổng số unit, giá mua, các khoản phân phối và các lần redemption của một investment product hoàn chỉnh. | `pdf005.php`<br/>`RT005.pdf.tpl` |

---

## 5. Loại B — Distributed Ledger (GoQuorum On-Chain)

Bây giờ hãy xem nhanh loại ledger còn lại trong OwnersBook+: **Distributed Ledger**.

### 1. Dữ liệu On-Chain là gì?
- **On-chain** nghĩa là dữ liệu được ghi vĩnh viễn vào các block trên một blockchain network.
- Chúng ta sử dụng một blockchain riêng tên **GoQuorum**, dựa trên Ethereum.
- Khi một investment fund được phát hành, một digital contract tạo security token theo standard **ERC-1400**.
- Khi investor giao dịch cổ phần, token được chuyển giữa các blockchain address.

### 2. Cách đọc dữ liệu On-Chain
- Blockchain node không thuận tiện cho việc hiển thị các web page nhanh.
- Một background worker đọc các blockchain block và copy transaction summary vào các database table thông thường như `bc_blocks`, `bc_tokens` và `block_tnx`.
- Chúng ta đã tạo một web viewer application riêng tên `st-bc-explorer`.
- Developer và compliance staff truy cập `st-bc-explorer` để tìm transaction hash, xem token holder và kiểm tra wallet balance.

### 3. Loại B liên quan đến Loại A như thế nào?
- Loại A xử lý kế toán pháp lý bằng đồng Yên Nhật, được lưu trong các database table truyền thống.
- Loại B xử lý security token kỹ thuật số, được lưu trên blockchain.
- Chúng là hai mặt của cùng một đồng tiền, nhưng sử dụng các software pipeline hoàn toàn riêng biệt.
- Các verification check về sau đảm bảo token trên blockchain khớp với số tiền được ghi trong statutory ledger.

---

## 6. Bảng tham chiếu API đầy đủ

Dưới đây là tất cả web endpoint giúp toàn bộ hệ sinh thái ledger hoạt động giữa các repository:

| Method | Endpoint | Repo | Mục đích | Request chính | Response chính | Auth | Mã lỗi chính |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `GET` | `/withdraw` | `common-front-api` | Hiển thị tài khoản ngân hàng đã đăng ký và số dư tiền mặt hiện tại của user | None (Header: Bearer Token) | Thông tin ngân hàng của user và số tiền mặt có thể rút | User Token | `E103-0001` (Please log in) |
| `POST` | `/withdraw/confirm` | `common-front-api` | Kiểm tra lại số tiền rút và tính phí chuyển khoản ngân hàng | Requested amount, password, device ID | Thông tin fee và one-time approval token | User Token | `E103-0504` (Missing info), `E103-0002` (Wrong password) |
| `POST` | `/withdraw/execute` | `common-front-api` | Gửi request rút tiền mặt cuối cùng vào queue của chúng ta | Amount, fees, scheduled date, SMS code | Simple success confirmation | User Token | `E103-0504` (Missing info), `E103-0003` (Not enough money) |
| `POST` | `/withdraw/cancel` | `common-front-api` | Hủy request rút tiền trước khi được xử lý | Request ID number and password | Simple success confirmation | User Token | `E103-0002` (Wrong password) |
| `GET` | `/payment/payin_bank_account` | `common-front-api` | Lấy số tài khoản ngân hàng ảo riêng của investor để nạp tiền | None (Header: Bearer Token) | Bank code, branch, and account number | User Token | `E103-0001` (Please log in) |
| `GET` | `/payments` | `common-front-api` | Lấy danh sách các ngân hàng thanh toán bên ngoài được hỗ trợ | None | List of available payment methods | User Token | `undetermined` |
| `GET` | `/payment/{PAYMENT_ID}` | `common-front-api` | Lấy connection setting cho payment bank đã chọn | Bank payment ID in the URL | Technical configuration for the bank | User Token | `undetermined` |
| `POST` | `/payment/nttd/bind` | `common-front-api` | Bank webhook xác nhận investor đã gửi tiền qua wire transfer | Bank notification payload | Success response to the bank | IP Whitelist / Webhook Secret | `undetermined` |
| `GET` | `/payment/activities` | `common-front-api` | Xem activity feed về các hoạt động nạp tiền và số dư gần đây | None | List of past deposit activities | User Token | `undetermined` |
| `GET` | `/electronic_delivery` | `common-front-api` | Liệt kê các PDF report chính thức mà user có thể xem | Filter parameters (date, read status) | List of reports with download links | User Token | `E103-0001` (Please log in) |
| `POST` | `/electronic_delivery/confirm` | `common-front-api` | Lưu việc investor đồng ý nhận report bằng phương thức điện tử | Document ID and version number | Simple success confirmation | User Token | `E103-0504` (Missing info), `E103-0278` (Invalid number) |
| `POST` | `/electronic_delivery/read` | `common-front-api` | Đánh dấu statutory report điện tử là đã mở và đã đọc | Document ID number | Document details and S3 storage path | User Token | `E103-0539` (Document not found) |
| `GET` | `/electronic_delivery/required_electronic` | `common-front-api` | Kiểm tra có tài liệu chưa đọc nào cần được chú ý ngay hay không | None | Yes/No flag and required document list | User Token | `undetermined` |
| `GET` | `/report_manage/` | `st-admin` | Xem và download các statutory ledger file nội bộ (RT004, RT005) | Folder browsing request | Web page directory list of downloadable PDFs | Admin Login | `403 Forbidden`, `404 Not Found` |
| `GET` | `/report_customer/` | `st-admin` | Xem và download các statutory ledger file của customer (RT003) | Folder browsing request | Web page directory list of downloadable PDFs | Admin Login | `403 Forbidden`, `404 Not Found` |
| `GET` | `/home` | `st-bc-explorer` | Mở dashboard chính hiển thị hoạt động blockchain | None | Web page with block and token statistics | Admin / Staff Login | `undetermined` |
| `GET` | `/token/{address}/erc1400_tnx_list` | `st-bc-explorer` | Hiển thị mọi giao dịch của một security token contract cụ thể | Token contract address in the URL | Web page listing token mints and transfers | Admin / Staff Login | `undetermined` |
| `GET` | `/token/{address}/owner_list` | `st-bc-explorer` | Hiển thị wallet address nào đang giữ unit của một token | Token contract address in the URL | Web page listing token holders and amounts | Admin / Staff Login | `undetermined` |
| `GET` | `/block/{block_number}/tnx_list` | `st-bc-explorer` | Hiển thị các giao dịch nằm trong một blockchain block cụ thể | Block number in the URL | Web page showing block details | Admin / Staff Login | `undetermined` |
| `GET` | `/wallet/{wallet_address}/...` | `st-bc-explorer` | Hiển thị tất cả token do một digital wallet cụ thể sở hữu | Wallet address in the URL | Web page with wallet token balances | Admin / Staff Login | `undetermined` |
| `GET` | `/tnx/{tnx_hash}` | `st-bc-explorer` | Tra cứu raw cryptographic transaction receipt | Transaction hash in the URL | Web page showing gas, sender, and receiver | Admin / Staff Login | `undetermined` |
| `POST` | `/search` | `st-bc-explorer` | Tìm kiếm bất kỳ block, transaction hoặc address nào trên blockchain | Search text entered in the search bar | Automatically jumps to the matching page | Admin / Staff Login | `undetermined` |

---

## 7. Điểm cần lưu ý và các lỗi thường gặp

1. **Holiday vẫn thành công một cách yên lặng**:
   - Nếu batch chạy vào thứ Bảy, Chủ nhật hoặc ngày nghỉ lễ quốc gia, nó xuất `OK(CLOSING)` rồi kết thúc mà không tạo PDF nào. Đây là hành vi có chủ đích, không phải lỗi.
2. **Batch hàng tháng chỉ chạy vào cuối tháng**:
   - Trading ledger hàng tháng (RT005) sẽ lập tức kết thúc với `OK(CLOSING)` nếu bạn chạy vào bất kỳ ngày nào không phải ngày cuối cùng của tháng.
3. **Bắt buộc phải có Database Configuration**:
   - Batch RT003 sẽ crash nếu thiếu các page threshold number trong configuration table `ESYS_CONFIG`.
4. **Permission của binary trong Docker container**:
   - PDF generator là một compiled binary executable. Nếu thiếu execution permission cho file, generator sẽ fail và tạo ra các file rỗng.
5. **Temporary Work File sẽ biến mất**:
   - Batch tự động dọn các local temporary folder ngay sau khi file được upload lên AWS S3. Nếu bạn tìm file local sau một lần chạy thành công, chúng đã bị xóa.
