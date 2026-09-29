# Tài liệu Đặc tả API: st-api (Fund Core API / Security Token Trading API)

## 1. Thông tin Dự án (Project Information)

- **Tên dự án (Project Name)**: `st-api` (`ST-FRONT-API`)
- **Ngôn ngữ chính (Primary Language)**: PHP 8.3 (được kiểm thử trong môi trường runtime PHP 8.3.16 trên container `st-app-pfund`)
- **Framework & Kiến trúc phần mềm (Framework & Software Architecture)**:
  - **Framework**: Brave Framework gọn nhẹ tùy biến (cấu trúc dạng MVC với `BraveDispatcher`, `BraveController`, `AppController`).
  - **Kiến trúc (Architecture)**: Kiến trúc phân tầng gồm HTTP Dispatcher -> Request Interceptor / Middlewares (`AppController`) -> Controller (`App\Controllers`) -> Service Layer (`App\Service`) -> Model Layer / Database Abstraction (`App\Models`, `BraveModel`).
  - **Engine Sổ cái & Kế toán (Ledger & Accounting Engine)**: `netsec/common-balance` (`CashAbleModel`, `CashRealBalanceModel`) sử dụng các phép toán số học độ chính xác tùy ý thông qua PHP `ext-bcmath` (nghiêm cấm tuyệt đối các phép tính số thực dấu phẩy động - floating-point - nhằm đảm bảo tính toàn vẹn tài chính).
  - **Tầng Session & Caching**: Redis (quản lý session OAuth2 Bearer token `TK:<company_seq>:<access_token>`, bộ đếm giới hạn tốc độ gọi API `RATE_LIMIT:<user_seq>:<window>`, token giao dịch chống giả mạo `ct:<token>`, bộ nhớ đệm preview đơn hàng `TMP_ORDER:<order_id>`).
  - **Reverse Proxy / Định tuyến (Routing)**: Nginx reverse proxy rewrite engine (`pfund-api.conf`) ánh xạ các đường dẫn RESTful sang query parameters của Brave `/index.php?c=<controller>&a=<action>`.
  - **Đo lường & Giám sát (Telemetry & Monitoring)**: Sentry SDK (`sentry/sdk: ^3.1`), Monolog (`monolog/monolog: ^2.3`), Fluentd logger (`fluent/logger: ^1.0`).
- **Cơ sở dữ liệu & Tầng truy cập dữ liệu (Database & Data Access Layer)**:
  - **Trừu tượng hóa (Abstraction)**: `adodb/adodb-php: 5.22.7` tích hợp qua `BraveMySQLBuilder`.
  - **Xử lý Multi-Tenancy**: Định tuyến database động dựa trên header request `X-Ci` (Company ID). Hệ thống tra cứu bảng `basis.BCO_COMPANY` với điều kiện `COMPANY_ID = :x_ci` để lấy `COMPANY_SEQ_NO` và `DB_NAME` (`{prefix}_pfund`), từ đó chuyển đổi kết nối database đang hoạt động tương ứng.
  - **Cụm Cơ sở dữ liệu (Database Cluster)**:
    - `basis`: Database hệ thống dùng chung chứa `BSYS_CONFIG`, `BSYS_MAINTENANCE`, `BSYS_MAINTENANCE_PERIODIC`, `BCO_COMPANY`.
    - `{prefix}_pfund`: Database giao dịch quỹ & Security Token chứa `PST_FUND`, `PST_BUY_ORDER`, `PST_PRICE`, `PST_CONTRACT_PDF`, `PST_CONTRACT_LOG`, `PST_PRI_ENTRY`, `PST_INCOME_TAX`, v.v.
    - `{prefix}`: Database core của công ty/khách hàng chứa `ECO_USER`, `ECO_DEVICE`, `ECO_TRADE_HIST`, `ECO_BASE_DATE`, `ECO_ELECTRONIC_DELIVERY`, `ECO_USER_INSIDER`.

---

## 2. Bảng Tổng hợp (Index)

| Tên API | Method | URL | Module | Trạng thái |
|---|---|---|---|---|
| [Tổng hợp tài sản (Asset Summary)](#1-tong-hop-tai-san-asset-summary) | GET | `/asset/summary` | Asset / Portfolio | Implemented |
| [Danh sách quỹ (Fund List)](#2-danh-sach-quy-fund-list) | GET | `/fund/list` | Fund | Implemented |
| [Chi tiết quỹ (Fund Detail)](#3-chi-tiet-quy-fund-detail) | GET | `/fund/detail` | Fund | Implemented |
| [Lịch sử đặt lệnh (Order History List)](#4-lich-su-dat-lenh-order-history-list) | POST | `/history/order/list` | History | Implemented |
| [Lịch sử khớp lệnh (Trade Execution History List)](#5-lich-su-khop-lenh-trade-execution-history-list) | POST | `/history/trade/list` | History | Implemented |
| [Lịch sử cổ tức & hoàn vốn (Dividend & Refund History List)](#6-lich-su-co-tuc--hoan-von-dividend--refund-history-list) | POST | `/history/dividend_refund/list` | History | Implemented |
| [Lịch sử quyết toán (Settlement History List)](#7-lich-su-quyet-toan-settlement-history-list) | POST | `/history/all/list` | History | Implemented |
| [Dữ liệu hỗ trợ quyết toán thuế (Income Tax Support Data)](#8-du-lieu-ho-tro-quyet-toan-thue-income-tax-support-data) | GET / POST | `/history/incometax` | History / Tax | Implemented |
| [Danh sách tài liệu hợp đồng (Contract Document List)](#9-danh-sach-tai-lieu-hop-dong-contract-document-list) | GET / POST | `/order/contract/list` | Order / Contract | Implemented |
| [Đọc tài liệu hợp đồng (Contract Document Read)](#10-doc-tai-lieu-hop-dong-contract-document-read) | GET / POST | `/order/contract/read` | Order / Contract | Implemented |
| [Đồng ý tài liệu hợp đồng (Contract Document Accept)](#11-dong-y-tai-lieu-hop-dong-contract-document-accept) | POST | `/order/contract/accept` | Order / Contract | Implemented |
| [Danh sách văn bản điều khoản (Text Document List)](#12-danh-sach-van-ban-dieu-khoan-text-document-list) | GET | `/order/textdocument/list` | Order / Text Document | Implemented |
| [Đồng ý văn bản điều khoản (Text Document Accept)](#13-dong-y-van-ban-dieu-khoan-text-document-accept) | POST | `/order/textdocument/accept` | Order / Text Document | Implemented |
| [Thông tin nhập lệnh (Order Input Information)](#14-thong-tin-nhap-lenh-order-input-information) | GET / POST | `/order/buy/input` | Order / Trading | Implemented |
| [Xác nhận lệnh (Order Confirmation)](#15-xac-nhan-lenh-order-confirmation) | POST | `/order/buy/confirm` | Order / Trading | Implemented |
| [Hoàn tất đặt lệnh (Order Execution Complete)](#16-hoan-tat-dat-lenh-order-execution-complete) | POST | `/order/buy/complete` | Order / Trading | Implemented |
| [Hủy lệnh (Order Cancellation)](#17-huy-lenh-order-cancellation) | POST | `/order/cancel` | Order / Trading | Implemented |
| [Xác nhận người nội bộ quỹ (Fund Insider Confirmation)](#18-xac-nhan-nguoi-noi-bo-quy-fund-insider-confirmation) | GET / POST | `/order/insiderconfirm` | Order / Compliance | Implemented |
| [Trang MyPage đăng ký ưu tiên (Priority Entry MyPage)](#19-trang-mypage-dang-ky-uu-tien-priority-entry-mypage) | GET | `/prientry/mypage` | Priority Entry | Implemented |
| [Chi tiết quỹ đăng ký ưu tiên (Priority Entry Fund Detail)](#20-chi-tiet-quy-dang-ky-uu-tien-priority-entry-fund-detail) | GET | `/prientry/fund/{FUND_CD}` | Priority Entry | Implemented |
| [Đăng ký tham gia bốc thăm ưu tiên (Priority Entry Application)](#21-dang-ky-tham-gia-boc-tham-uu-tien-priority-entry-application) | POST | `/prientry/entry/{FUND_CD}` | Priority Entry | Implemented |
| [Cập nhật số lượng đăng ký ưu tiên (Priority Entry Update Quantity)](#22-cap-nhat-so-luong-dang-ky-uu-tien-priority-entry-update-quantity) | POST | `/prientry/update/{FUND_CD}` | Priority Entry | Implemented |
| [Hủy đăng ký ưu tiên (Priority Entry Cancellation)](#23-huy-dang-ky-uu-tien-priority-entry-cancellation) | POST | `/prientry/cancel/{FUND_CD}` | Priority Entry | Implemented |
| [Webhook thông báo gửi email SendGrid (SendGrid Webhook Notification)](#24-webhook-thong-bao-gui-email-sendgrid-sendgrid-webhook-notification) | POST | `/webhook/transfer-notification/sendgrid/{x-ci}` | Notification / Webhook | Implemented |
| [Proxy hình ảnh icon S3 (S3 Icon Image Proxy)](#25-proxy-hinh-anh-icon-s3-s3-icon-image-proxy) | GET | `/s3/{x-ci}/{icon-key}` | Media / Proxy | Implemented |

---

## 3. Nội dung Chi tiết: Thông tin từng API (Main Content: API Details)

---

### 1. Tổng hợp tài sản (Asset Summary)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Asset Summary (`ST-API-AST-001`)
- **Controller - Service**: `App\Controllers\AssetController::summary` -> `App\Service\FundBalanceService::summary`
- **Tệp nguồn / Class**: [`App/Controllers/AssetController.php::summary`](/st-api/App/Controllers/AssetController.php#L10-L14)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Lấy snapshot tổng hợp danh mục tài sản của nhà đầu tư đang đăng nhập, bao gồm tiền gửi/tiền ký quỹ (預り金 - *Azukarikin*), số tiền khả dụng để mua (買付可能額 - *Kaitsuke Kanogaku*), tổng giá trị định giá các chứng khoán số Security Token đang nắm giữ (ST評価額合計), giá trị các giao dịch mua chưa quyết toán (受渡前の買付額合計), và chi tiết phân bổ danh mục tài sản theo từng quỹ.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `GET`
- **URL**: `/asset/summary` (Rewrite nội bộ: `/index.php?c=asset&a=summary`)
- **Môi trường**: Tất cả môi trường (Local, Dev, Staging, Production)
- **Phiên bản**: v1

#### Xác thực & Phân quyền (Authentication & Authorization)
- **Loại xác thực (Auth Type)**: Bearer Token (OAuth2 access token được cấp bởi service `passport`)
- **Quyền hạn (Permissions)**: Session client đã được xác thực trong Redis (`TK:<COMPANY_SEQ_NO>:<token>`)
- **Headers đặc thù**:
  - `Authorization`: `Bearer <token>` (Bắt buộc)
  - `X-Ci`: Mã định danh công ty (Company ID, ví dụ: `sgDj8sHr`) (Bắt buộc)
  - `X-Pf`: Nền tảng client (`Web`, `iOS`, `Android`) (Bắt buộc)
  - `X-Sv`: Mã chuỗi service (Service sequence number) (Bắt buộc)
  - `X-Du`: UUID thiết bị client khớp với `ECO_DEVICE.CREDIBLE_FLG = '1'` (Bắt buộc)
- **Giới hạn tốc độ (Rate Limits)**: Bộ đếm trượt Redis `RATE_LIMIT:<USER_SEQ_NO>:<window>` (Mặc định: tối đa 30 requests trong 60 giây).

#### Chi tiết Request (Request Details)
- **Headers**:
  ```http
  Authorization: Bearer <access_token>
  X-Ci: sgDj8sHr
  X-Pf: Web
  X-Sv: 1
  X-Du: 12345678-1234-4234-8234-1234567890ab
  ```
- **Query Parameters**: Không có (hoặc tham số tùy chọn `TYPE=ASSETS` / `TYPE=BALANCE` được sử dụng bởi bộ lọc frontend)
- **Body Schema**: Không có
- **Payload mẫu**: Không có

#### Chi tiết Response (Response Details)
- **Status Codes**: `200 OK` (HTTP status code luôn trả về 200 ngay cả khi có lỗi logic nghiệp vụ, bao bọc bởi `STATUS: "OK"` hoặc `"NG"`)
- **Schema**:
  ```json
  {
    "STATUS": "OK",
    "ERROR": null,
    "DATA": {
      "DEPOSIT_AMOUNT": "string (number)",
      "BUYABLE_AMOUNT": "string (number)",
      "EVALUATION_AMOUNT_SETTLEMENT_TOTAL": "string (number)",
      "BUY_AMOUNT_BEFORE_SETTLEMENT_TOTAL": "string (number)",
      "ASSETS_TOTAL": "string (number)",
      "DIVIDEND_SUM_AMOUNT_TRADE_TOTAL": "string (number)",
      "FUND_REAL_BALANCE_LIST": [
        {
          "SHOW_FLAG": "string (0|1)",
          "DATA_TYPE": "string (1: owned, 2: un-settled buy)",
          "FUND_CD": "string",
          "FUND_NM": "string",
          "FAILED_FLG": "integer",
          "AMOUNT": "string (number)",
          "DIVIDEND_SUM_AMOUNT_TRADE": "string (number)"
        }
      ]
    }
  }
  ```
- **Mã lỗi nghiệp vụ (Business Error Codes)**: `unauth` (401), `xdu_credible_error`, `rate_limit_msg`, `no_account_number_verified_error`
- **Response mẫu**:
  ```json
  {
    "STATUS": "OK",
    "ERROR": null,
    "DATA": {
      "DEPOSIT_AMOUNT": "1500000",
      "BUYABLE_AMOUNT": "1200000",
      "EVALUATION_AMOUNT_SETTLEMENT_TOTAL": "3000000",
      "BUY_AMOUNT_BEFORE_SETTLEMENT_TOTAL": "0",
      "ASSETS_TOTAL": "4200000",
      "DIVIDEND_SUM_AMOUNT_TRADE_TOTAL": "85000",
      "FUND_REAL_BALANCE_LIST": [
        {
          "SHOW_FLAG": "1",
          "DATA_TYPE": "1",
          "FUND_CD": "FND001",
          "FUND_NM": "Tokyo Logistics ST Fund 01",
          "FAILED_FLG": 0,
          "AMOUNT": "3000000",
          "DIVIDEND_SUM_AMOUNT_TRADE": "85000"
        }
      ]
    }
  }
  ```

#### Luồng xử lý & Giao dịch (Processing Flow & Transactions)
- **Luồng xử lý**:
  1. Base controller kiểm tra chế độ bảo trì, xác thực Bearer token, rate limit, thiết bị đáng tin cậy và trạng thái tài khoản pháp nhân.
  2. Truy vấn `BaseDateModel` để lấy ngày cơ sở giao dịch hiện tại (`CURRENT_D`).
  3. Truy vấn `CashAbleModel` (`netsec/common-balance`) để lấy số dư tiền gửi và tiền khả dụng mua.
  4. Truy vấn `FundRealBalanceModel` lấy danh sách quỹ nắm giữ và tính toán định giá ST dựa trên giá từ `PriceModel` qua `AssetHelper::fixForSettlement`.
  5. Tính toán tổng định giá danh mục đầu tư bằng thư viện `ext-bcmath`.
- **Ranh giới giao dịch (Transaction Boundaries)**: Chỉ đọc (Read-only), không mở database transaction.
- **Tính lũy đẳng (Idempotency)**: Idempotent (`GET`).

#### Tác dụng phụ (Side Effects)
- **Caching**: Cập nhật thời gian hết hạn của token trong Redis (`TK:...` gia hạn thêm 3600 giây) và tăng bộ đếm rate limit.
- **Logging**: Ghi log truy cập qua Monolog/Fluentd.

#### Phụ thuộc (Dependencies)
- **Internal Services**: `FundBalanceService`, `CashAbleModel` (`netsec/common-balance`), `PriceModel`, `TradeHistModel`.
- **External Services**: Redis, MySQL (`basis`, `{prefix}`, `{prefix}_pfund`).
- **APIs liên quan**: `/fund/list`, `/order/buy/input`.

---

### 2. Danh sách quỹ (Fund List)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Fund List (`ST-API-FND-001`)
- **Controller - Service**: `App\Controllers\FundController::List` -> `App\Service\FundService::fundListForFind` / `FundModel::fundListSmall`
- **Tệp nguồn / Class**: [`App/Controllers/FundController.php::List`](/st-api/App/Controllers/FundController.php#L13-L58)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Lấy danh sách các quỹ đầu tư theo điều kiện lọc và tìm kiếm. Trong mã nguồn hiện tại, validation bắt buộc `TYPE=simple` (danh sách rút gọn mã và tên dùng cho dropdown) hoặc `TYPE=FIND` (tìm kiếm đầy đủ với bộ lọc trạng thái quỹ, dải lợi suất, tag).

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `GET`
- **URL**: `/fund/list` (Rewrite nội bộ: `/index.php?c=fund&a=list`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Xác thực & Phân quyền (Authentication & Authorization)
- **Loại xác thực (Auth Type)**: Bearer Token
- **Quyền hạn (Permissions)**: Session client đã được xác thực
- **Headers đặc thù**: `Authorization`, `X-Ci`, `X-Pf`, `X-Sv`, `X-Du`
- **Giới hạn tốc độ (Rate Limits)**: Giới hạn tiêu chuẩn (30 requests / 60 giây).

#### Chi tiết Request (Request Details)
- **Headers**: Các header chuẩn của `AppController`.
- **Query Parameters**:
  - `TYPE`: `string` (Bắt buộc; giá trị hợp lệ hiện tại: `simple` hoặc `FIND`)
  - `CONDITION_LIST`: `array` (Tùy chọn; áp dụng khi `TYPE=FIND`, hỗ trợ lọc theo trạng thái, ngày đặt lệnh, tag, lợi suất mục tiêu)
- **URL mẫu**: `/fund/list?TYPE=FIND`

#### Chi tiết Response (Response Details)
- **Status Codes**: `200 OK`
- **Schema**:
  ```json
  {
    "STATUS": "OK",
    "ERROR": null,
    "DATA": {
      "FUND_LIST": [
        {
          "FUND_CD": "string",
          "FUND_NM": "string",
          "TARGET_RATE": "string",
          "ORDER_START_DT": "string (datetime)",
          "ORDER_END_DT": "string (datetime)",
          "STATUS_TEXT": "string",
          "TOTAL_INVESTMENT_AMOUNT": "string (number)"
        }
      ]
    }
  }
  ```
- **Mã lỗi nghiệp vụ (Business Error Codes)**: `system_error` (bị ném ra nếu `TYPE` bị bỏ trống hoặc không phải là `simple` hay `FIND`).

#### Luồng xử lý & Giao dịch (Processing Flow & Transactions)
- **Luồng xử lý**:
  1. Kiểm tra tính hợp lệ của tham số `TYPE`.
  2. Nếu `TYPE == 'simple'`, gọi `FundModel::fundListSmall()` trả về danh sách cặp mã và tên rút gọn.
  3. Nếu `TYPE == 'FIND'`, gọi `FundService::fundListForFind($condition_list)`, tổng hợp thông tin tag quỹ, số lượng đăng ký và trạng thái phát hành.
- **Ranh giới giao dịch**: Chỉ đọc (Read-only).
- **Tính lũy đẳng**: Idempotent.

#### Tác dụng phụ & Phụ thuộc (Side Effects & Dependencies)
- **Logging**: Ghi access log.
- **Dependencies**: `PST_FUND`, `PST_FUND_TAG`, `PST_FUND_APPRICATION`.

---

### 3. Chi tiết quỹ (Fund Detail)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Fund Detail (`ST-API-FND-002`)
- **Controller - Service**: `App\Controllers\FundController::Detail` -> `App\Service\FundService::detail` & `PrientryService::getEntryElected`
- **Tệp nguồn / Class**: [`App/Controllers/FundController.php::Detail`](/st-api/App/Controllers/FundController.php#L60-L72)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Cung cấp thông tin chi tiết toàn diện của một quỹ cụ thể, bao gồm tổng quan đầu tư, lịch biểu tài sản, tỷ suất sinh lời, điều khoản hợp đồng, trạng thái đăng ký bốc thăm, và trạng thái trúng thăm quyền mua ưu tiên (*Elected status*) của người dùng đang đăng nhập.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `GET`
- **URL**: `/fund/detail` (Rewrite nội bộ: `/index.php?c=fund&a=detail`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Xác thực & Phân quyền (Authentication & Authorization)
- **Loại xác thực (Auth Type)**: Bearer Token
- **Quyền hạn (Permissions)**: Session client đã được xác thực
- **Headers đặc thù**: `Authorization`, `X-Ci`, `X-Pf`, `X-Sv`, `X-Du`
- **Giới hạn tốc độ (Rate Limits)**: Giới hạn tiêu chuẩn.

#### Chi tiết Request (Request Details)
- **Query Parameters**:
  - `FUND_CD`: `string` (Bắt buộc; mã định danh quỹ)
- **URL mẫu**: `/fund/detail?FUND_CD=FND001`

#### Chi tiết Response (Response Details)
- **Status Codes**: `200 OK`
- **Schema**:
  ```json
  {
    "STATUS": "OK",
    "ERROR": null,
    "DATA": {
      "FUND": {
        "FUND_CD": "string",
        "FUND_NM": "string",
        "TARGET_RATE": "string",
        "UNIT_PRICE": "string",
        "MIN_QTY": "string",
        "UNIT_QTY": "string",
        "MAX_QTY": "string",
        "PERIOD": "string (PRIMARY|SECONDARY|PRIENTRY|PRE_SECONDARY)",
        "BUYABLE_FLG": "integer",
        "SELL_FLG": "integer",
        "ORDER_START_DT": "string",
        "ORDER_END_DT": "string",
        "ESTIMATED_DIVIDEND": "string"
      },
      "PRI_ENTRY_ELECTED": {
        "ELECTION_STATUS": "integer",
        "ORDER_QTY": "string"
      }
    }
  }
  ```
- **Mã lỗi nghiệp vụ (Business Error Codes)**: `fund_not_exist`

#### Luồng xử lý & Phụ thuộc (Processing Flow & Dependencies)
- **Luồng xử lý**:
  1. Kiểm tra sự tồn tại của quỹ và cờ công khai `OPEN_FLG = 1`.
  2. Xác định giai đoạn vòng đời hiện tại của quỹ (Phát hành sơ cấp, Đăng ký bốc thăm, hoặc Giao dịch thứ cấp) qua `FundHelper::fixFund`.
  3. Kiểm tra trạng thái trúng thăm phân bổ ưu tiên qua `PrientryService` (`PST_PRI_ENTRY_ELECTED`).
- **Dependencies**: `PST_FUND`, `PST_PRI_ENTRY_ELECTED`, `PST_PRICE`.

---

### 4. Lịch sử đặt lệnh (Order History List)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Order History List (`ST-API-HIS-001`)
- **Controller - Service**: `App\Controllers\HistoryController::OrderList`
- **Tệp nguồn / Class**: [`App/Controllers/HistoryController.php::OrderList`](/st-api/App/Controllers/HistoryController.php#L18-L77)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Liệt kê tất cả các lệnh mua/bán chưa khớp hoặc có thể hủy của người dùng hiện tại, tính toán cờ cho phép hủy lệnh dựa trên ngày hiện tại và thời hạn chốt sổ giao dịch trong ngày (cut-off deadline).

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `POST` (nhận payload điều kiện lọc qua JSON body)
- **URL**: `/history/order/list` (Rewrite nội bộ: `/index.php?c=history&a=orderList`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Xác thực & Phân quyền (Authentication & Authorization)
- **Loại xác thực (Auth Type)**: Bearer Token
- **Quyền hạn (Permissions)**: Session client đã được xác thực
- **Headers đặc thù**: `Authorization`, `X-Ci`, `X-Pf`, `X-Sv`, `X-Du`
- **Giới hạn tốc độ (Rate Limits)**: Giới hạn tiêu chuẩn.

#### Chi tiết Request (Request Details)
- **Body Schema**:
  ```json
  {
    "ORDER_D_TO": "string (YYYY-MM-DD, tùy chọn, mặc định là CURRENT_D)",
    "FUND_CD": "string (tùy chọn)",
    "ORDER_OBJECT": "array of string (tùy chọn: BUY, TRADE-BUY, TRADE-SELL)"
  }
  ```
- **Payload mẫu**:
  ```json
  {
    "ORDER_D_TO": "2026-09-09"
  }
  ```

#### Chi tiết Response (Response Details)
- **Status Codes**: `200 OK`
- **Schema**:
  ```json
  {
    "STATUS": "OK",
    "ERROR": null,
    "DATA": {
      "QUERY": "object",
      "ORDER_LIST": [
        {
          "ORDER_NUM": "string",
          "FUND_CD": "string",
          "FUND_NM": "string",
          "BUYSELL": "string (1: sell, 2: buy)",
          "BUYADD_FLG": "string (0: primary, 2: secondary)",
          "ORDER_QTY": "string",
          "ORDER_PER_PRICE": "string",
          "ORDER_AMOUNT": "string",
          "ORDER_STATUS": "integer (0: awaiting settlement, 10: cancelled)",
          "ORDER_STATUS_TEXT": "string",
          "ORDER_OBJECT_TEXT": "string (募集買付, 買付, 売却, 取消済)",
          "CANCEL_VALID": "boolean"
        }
      ],
      "FUND_OPTIONS": "array",
      "ORDER_OBJECT_TYPES": "array"
    }
  }
  ```

#### Luồng xử lý & Phụ thuộc (Processing Flow & Dependencies)
- **Luồng xử lý**: Lấy danh sách lệnh chưa khớp từ `PST_BUY_ORDER`, join với `PST_FUND`, kiểm tra deadline hủy lệnh (`pfund.order.deadline`) trong `ESYS_CONFIG` qua `BuyService::orderCheckCancelValid` / `TradeService::orderCheckCancelValid`.
- **Dependencies**: `PST_BUY_ORDER`, `PST_BASE_DATE`, `ESYS_CONFIG`.

---

### 5. Lịch sử khớp lệnh (Trade Execution History List)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Trade Execution History List (`ST-API-HIS-002`)
- **Controller - Service**: `App\Controllers\HistoryController::TradeList`
- **Tệp nguồn / Class**: [`App/Controllers/HistoryController.php::TradeList`](/st-api/App/Controllers/HistoryController.php#L79-L157)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Truy xuất lịch sử các giao dịch đã khớp (約定履歴 - *Yakujo Rireki*), bao gồm các giao dịch mua phát hành sơ cấp, mua/bán thứ cấp, chuyển kho lưu ký (入庫/出庫), và hoàn vốn/đáo hạn.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `POST`
- **URL**: `/history/trade/list` (Rewrite nội bộ: `/index.php?c=history&a=tradeList`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Chi tiết Request & Response (Request & Response Details)
- **Body Schema**:
  ```json
  {
    "D_TYPE": "string (TRADE_D | SETTLEMENT_D, mặc định là TRADE_D)",
    "TRADE_OBJECT": "array of string (BUY, TRADE-BUY, TRADE-SELL, IN-DEPOT, OUT-DEPOT, REFUND)",
    "D_FROM": "string (YYYY-MM-DD, mặc định 1 tháng trước)",
    "D_TO": "string (YYYY-MM-DD, mặc định là CURRENT_D)"
  }
  ```
- **Response Schema**:
  ```json
  {
    "STATUS": "OK",
    "ERROR": null,
    "DATA": {
      "TRADE_LIST": [
        {
          "TRADE_NUM": "string",
          "FUND_CD": "string",
          "FUND_NM": "string",
          "SUMMARY_TYPE": "integer",
          "SUMMARY_TYPE_TEXT": "string",
          "TRADE_D": "string (YYYY-MM-DD)",
          "SETTLEMENT_D": "string (YYYY-MM-DD)",
          "QTY": "string",
          "PER_PRICE": "string",
          "TRADE_AMOUNT": "string",
          "ACCOUNT_TYPE_TEXT": "一般"
        }
      ]
    }
  }
  ```
- **Dependencies**: `PST_ALL_TRADE_HIST`, `PST_BASE_DATE`, `PST_FUND`.

---

### 6. Lịch sử cổ tức & hoàn vốn (Dividend & Refund History List)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Dividend & Refund History List (`ST-API-HIS-003`)
- **Controller - Service**: `App\Controllers\HistoryController::DividendRefundList`
- **Tệp nguồn / Class**: [`App/Controllers/HistoryController.php::DividendRefundList`](/st-api/App/Controllers/HistoryController.php#L159-L210)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Lấy dữ liệu lịch sử nhận chi trả lợi tức/cổ tức (分配金 - *Bunpaikin*) và tiền hoàn vốn/đáo hạn (償還金 - *Shokankin*) của khách hàng.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `POST`
- **URL**: `/history/dividend_refund/list` (Rewrite nội bộ: `/index.php?c=history&a=dividendRefundList`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Chi tiết Request & Response (Request & Response Details)
- **Body Schema**:
  ```json
  {
    "TRADE_OBJECT": "array of string (DIVIDEND, REFUND)",
    "D_FROM": "string (YYYY-MM-DD)",
    "D_TO": "string (YYYY-MM-DD)"
  }
  ```
- **Response Schema**: Trả về mảng `TRADE_LIST` chứa `SUMMARY_TYPE`, `PAYMENT_D`, `QTY`, `TRADE_AMOUNT`, `TRADE_PER_PRICE` và số thuế bị khấu trừ tại nguồn.
- **Dependencies**: `PST_ALL_TRADE_HIST`, `PST_BASE_DATE`.

---

### 7. Lịch sử quyết toán (Settlement History List)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Settlement History List (`ST-API-HIS-004`)
- **Controller - Service**: `App\Controllers\HistoryController::AllList`
- **Tệp nguồn / Class**: [`App/Controllers/HistoryController.php::AllList`](/st-api/App/Controllers/HistoryController.php#L215-L284)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Trả về sổ cái quyết toán thống nhất theo trình tự thời gian (精算履歴 - *Seisan Rireki*), kết hợp cả giao dịch chứng khoán, tiền cổ tức, tiền hoàn vốn, nạp tiền ngân hàng và rút tiền có ngày thanh toán nhỏ hơn hoặc bằng ngày cơ sở hiện tại.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `POST`
- **URL**: `/history/all/list` (Rewrite nội bộ: `/index.php?c=history&a=allList`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Chi tiết Request & Response (Request & Response Details)
- **Body Schema**:
  ```json
  {
    "TRADE_OBJECT": "array of string (BUY, TRADE-BUY, TRADE-SELL, DIVIDEND, REFUND, IN, OUT)",
    "D_FROM": "string (YYYY-MM-DD)",
    "D_TO": "string (YYYY-MM-DD)"
  }
  ```
- **Chi tiết Response**: Trả về danh sách `TRADE_LIST` được đính kèm các chỉ thị giao diện UI như `COLOR` (ví dụ: `#FFBBC3` cho lệnh mua, `#A0CAFA` cho lệnh bán/hoàn vốn, `#FFEE99` cho nạp/rút tiền) và `SUMMARY_TYPE_TEXT`.
- **Dependencies**: `ECO_TRADE_HIST`, `ECO_BASE_DATE`.

---

### 8. Dữ liệu hỗ trợ quyết toán thuế (Income Tax Support Data)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Income Tax Support Data (`ST-API-HIS-005`)
- **Controller - Service**: `App\Controllers\HistoryController::Incometax`
- **Tệp nguồn / Class**: [`App/Controllers/HistoryController.php::Incometax`](/st-api/App/Controllers/HistoryController.php#L289-L547)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Cung cấp dữ liệu tính thuế thu nhập phục vụ cho việc tự khai quyết toán thuế hàng năm tại Nhật Bản (確定申告サポート資料 - *Kakutei Shinkoku Support Shiryo*). Tổng hợp thu nhập cổ tức hàng năm, lãi/lỗ chuyển nhượng từ việc bán ST (譲渡損益), số tiền hoàn vốn, thu nhập vãng lai/khác (雑所得), số tiền thuế khấu trừ tại nguồn và thông tin địa chỉ đơn vị chi trả.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `GET` / `POST`
- **URL**: `/history/incometax` (Rewrite nội bộ: `/index.php?c=history&a=incometax`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Chi tiết Request (Request Details)
- **Parameters**: `YEAR`: `string` (Tùy chọn, định dạng 4 chữ số, ví dụ: `2025`; mặc định lấy năm đã được xử lý gần nhất)

#### Chi tiết Response (Response Details)
- **Response Schema**:
  ```json
  {
    "STATUS": "OK",
    "ERROR": null,
    "DATA": {
      "CONDITION": {
        "SELECTED_YEAR": "2025",
        "YEAR_LIST": ["2025", "2024"],
        "START_D": "2025-01-01",
        "DIV_END_D": "2025-12-31",
        "PRO_END_D": "2025-12-31",
        "REF_END_D": "2025-12-31"
      },
      "OTHERS_AMOUNT": {
        "OTHERS_AMOUNT_TOTAL": "string (number)",
        "PROFIT_DIVIDEND_AMOUNT_TOTAL": "string (number)",
        "TAX_TOTAL": "string (number)",
        "ASSIGNMENT_AMOUNT_TOTAL": "string (number)"
      },
      "DIVIDEND_AMOUNT": {
        "DIVIDEND_AMOUNT_TOTAL": "string (number)",
        "PROFIT_DIVIDEND_AMOUNT_TOTAL": "string (number)",
        "EXTRA_DIVIDEND_AMOUNT_TOTAL": "string (number)",
        "TAX_TOTAL": "string (number)",
        "LOSS_AMOUNT_TOTAL": "string (number)"
      },
      "DIVIDEND_DETAIL_SUM_LIST": "array",
      "DIVIDEND_DETAIL_LIST": "array",
      "GROSS_PROFIT": "object",
      "GROSS_PROFIT_SUM_LIST": "array",
      "GROSS_PROFIT_LIST": "array",
      "REFUND_AMOUNT": "object",
      "REFUND_DETAIL_LIST": "array",
      "SUPPLIER_INFO": {
        "NAME": "string",
        "ADDRESS": "string"
      }
    }
  }
  ```
- **Dependencies**: `PST_INCOME_TAX`, `ESYS_CONFIG` (`daybatch.company.print_self`, `daybatch.company.printinfo01`).

---

### 9. Danh sách tài liệu hợp đồng (Contract Document List)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Contract Document List (`ST-API-ORD-001`)
- **Controller - Service**: `App\Controllers\OrderController::ContractList` -> `App\Service\ContractService::contractList`
- **Tệp nguồn / Class**: [`App/Controllers/OrderController.php::ContractList`](/st-api/App/Controllers/OrderController.php#L15-L26)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Lấy danh sách các tài liệu công bố theo luật định dưới dạng điện tử (目論見書 - Bản cáo bạch, 契約締結前交付書面 - Tài liệu trước hợp đồng, 契約書 - Hợp đồng đầu tư) cho một quỹ cụ thể, chi tiết về phiên bản, loại tài liệu và trạng thái khách hàng đã đồng ý hay chưa.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `GET` / `POST`
- **URL**: `/order/contract/list` (Rewrite nội bộ: `/index.php?c=order&a=contractList`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Chi tiết Request (Request Details)
- **Parameters**: `FUND_CD`: `string` (Bắt buộc)

#### Chi tiết Response (Response Details)
- **Schema**:
  ```json
  {
    "STATUS": "OK",
    "ERROR": null,
    "DATA": {
      "CONTRACT_LIST": [
        {
          "CONTRACT_NUM": "string",
          "CONTRACT_TYPE": "string (1: Prospectus, 2: Pre-contract, 3: Contract)",
          "CONTRACT_VERSION": "string",
          "CONTRACT_PDF_PATH": "string",
          "AGREE_FLG": "string (0: not agreed, 1: agreed)"
        }
      ]
    }
  }
  ```
- **Mã lỗi nghiệp vụ (Business Error Codes)**: `fund_not_exist`
- **Dependencies**: `PST_CONTRACT_PDF`, `PST_CONTRACT_LOG`, `PST_FUND`.

---

### 10. Đọc tài liệu hợp đồng (Contract Document Read)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Contract Document Read (`ST-API-ORD-002`)
- **Controller - Service**: `App\Controllers\OrderController::ContractRead` -> `App\Service\ContractService::contractRead`
- **Tệp nguồn / Class**: [`App/Controllers/OrderController.php::ContractRead`](/st-api/App/Controllers/OrderController.php#L28-L37)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Tạo URL tạm thời có chữ ký (Pre-signed URL) từ AWS S3 để tải/xem tài liệu PDF hợp đồng và lưu vết người dùng đã mở/đọc tài liệu vào Redis.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `GET` / `POST`
- **URL**: `/order/contract/read` (Rewrite nội bộ: `/index.php?c=order&a=contractRead`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Chi tiết Request (Request Details)
- **Parameters**:
  - `FUND_CD`: `string` (Bắt buộc)
  - `CONTRACT_TYPE`: `string` (Bắt buộc; `1`, `2`, hoặc `3`)
  - `CONTRACT_VERSION`: `string` (Bắt buộc)

#### Chi tiết Response (Response Details)
- **Schema**:
  ```json
  {
    "STATUS": "OK",
    "ERROR": null,
    "DATA": {
      "CONTRACT_NUM": "string",
      "CONTRACT_TYPE": "string",
      "CONTRACT_VERSION": "string",
      "CONTRACT_PDF_NAME": "string",
      "CONTRACT_PDF_URL": "string (AWS S3 Pre-signed URL, valid 3600s)"
    }
  }
  ```
- **Mã lỗi nghiệp vụ (Business Error Codes)**: `pdf_not_exist`

#### Tác dụng phụ (Side Effects)
- **Caching**: Thiết lập hash Redis `CR:<CLIENT_ID>:<CONTRACT_NUM>` với key `<CONTRACT_VERSION>` lưu trữ metadata tài liệu và TTL là 24 giờ (86.400 giây).

---

### 11. Đồng ý tài liệu hợp đồng (Contract Document Accept)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Contract Document Accept (`ST-API-ORD-003`)
- **Controller - Service**: `App\Controllers\OrderController::ContractAccept` -> `App\Service\ContractService::contractAccept`
- **Tệp nguồn / Class**: [`App/Controllers/OrderController.php::ContractAccept`](/st-api/App/Controllers/OrderController.php#L39-L48)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Xác nhận sự đồng thuận/chấp thuận theo luật định của nhà đầu tư đối với văn bản công bố. Xác minh rằng tài liệu đã thực sự được mở và đọc qua Redis trước khi lưu trữ bản ghi đồng ý.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `POST`
- **URL**: `/order/contract/accept` (Rewrite nội bộ: `/index.php?c=order&a=contractAccept`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Chi tiết Request (Request Details)
- **Body Schema**:
  ```json
  {
    "FUND_CD": "string (Bắt buộc)",
    "CONTRACT_TYPE": "string (Bắt buộc: 1, 2, hoặc 3)",
    "CONTRACT_VERSION": "string (Bắt buộc)"
  }
  ```

#### Chi tiết Response (Response Details)
- **Schema**:
  ```json
  {
    "STATUS": "OK",
    "ERROR": null,
    "DATA": []
  }
  ```
- **Mã lỗi nghiệp vụ (Business Error Codes)**: `fund_not_exist`, `pdf_not_exist`, `pdf_not_read`, `system_error`

#### Luồng xử lý & Giao dịch (Processing Flow & Transactions)
- **Luồng xử lý**:
  1. Kiểm tra key Redis `CR:<CLIENT_ID>:<CONTRACT_NUM>` ứng với `<CONTRACT_VERSION>`. Ném ngoại lệ `pdf_not_read` nếu chưa đọc.
  2. Đối với loại hợp đồng `3` (Hợp đồng hợp danh ẩn danh), đảm bảo người dùng chưa từng ký một phiên bản khác trước đó.
  3. Bắt đầu transaction database (`AppModel::beginTransaction()`).
  4. Thêm bản ghi xác nhận vào `PST_CONTRACT_LOG`.
  5. Ghi nhận việc giao kết điện tử theo luật vào `ECO_ELECTRONIC_DELIVERY`.
  6. Commit transaction database.
- **Ranh giới giao dịch**: Đảm bảo tính nguyên tử (atomic) trên cả hai bảng `PST_CONTRACT_LOG` và `ECO_ELECTRONIC_DELIVERY`.

---

### 12. Danh sách văn bản điều khoản (Text Document List)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Text Document List (`ST-API-ORD-004`)
- **Controller - Service**: `App\Controllers\OrderController::TextdocumentList` -> `App\Service\TextDocumentService::textdocumentList`
- **Tệp nguồn / Class**: [`App/Controllers/OrderController.php::TextdocumentList`](/st-api/App/Controllers/OrderController.php#L50-L61)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Lấy danh sách các điều khoản cam kết dạng văn bản bắt buộc nhà đầu tư phải xác nhận trước khi gửi lệnh giao dịch.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `GET`
- **URL**: `/order/textdocument/list` (Rewrite nội bộ: `/index.php?c=order&a=textdocumentList`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Chi tiết Request & Response (Request & Response Details)
- **Parameters**: `FUND_CD`: `string` (Bắt buộc)
- **Response Schema**:
  ```json
  {
    "STATUS": "OK",
    "ERROR": null,
    "DATA": {
      "TEXTDOCUMENT_LIST": [
        {
          "SEQ_NO": "string",
          "FUND_CD": "string",
          "TITLE": "string",
          "CONTENT": "string (được format với thẻ HTML br)",
          "AGREE_FLG": "string"
        }
      ]
    }
  }
  ```
- **Dependencies**: `PST_TEXT_DOCUMENT`, `PST_TEXT_DOCUMENT_LOG`.

---

### 13. Đồng ý văn bản điều khoản (Text Document Accept)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Text Document Accept (`ST-API-ORD-005`)
- **Controller - Service**: `App\Controllers\OrderController::TextdocumentAccept` -> `App\Service\TextDocumentService::textDocumentAccept`
- **Tệp nguồn / Class**: [`App/Controllers/OrderController.php::TextdocumentAccept`](/st-api/App/Controllers/OrderController.php#L63-L72)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Ghi nhận sự đồng thuận của nhà đầu tư đối với các văn bản điều khoản cam kết dạng văn bản.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `POST`
- **URL**: `/order/textdocument/accept` (Rewrite nội bộ: `/index.php?c=order&a=textdocumentAccept`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Chi tiết Request (Request Details)
- **Body Schema**:
  ```json
  {
    "FUND_CD": "string (Bắt buộc)",
    "TEXTDOCUMENT_LIST": [
      {
        "TEXT_DOCUMENT_SEQ_NO": "string (Bắt buộc)"
      }
    ]
  }
  ```

#### Chi tiết Response (Response Details)
- **Schema**: `{"STATUS": "OK", "ERROR": null, "DATA": []}`
- **Luồng xử lý**: Khởi tạo database transaction, ghi bản ghi vào `PST_TEXT_DOCUMENT_LOG`, commit.

---

### 14. Thông tin nhập lệnh (Order Input Information)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Order Input Information (`ST-API-ORD-006`)
- **Controller - Service**: `App\Controllers\OrderController::BuyInput` -> `BuyService::buyInputBuy` / `TradeService::tradeBuyInput` / `TradeService::tradeSellInput`
- **Tệp nguồn / Class**: [`App/Controllers/OrderController.php::BuyInput`](/st-api/App/Controllers/OrderController.php#L74-L104)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Chuẩn bị dữ liệu hiển thị cho màn hình nhập lệnh đặt mua sơ cấp (`BUY`), người trúng thăm ưu tiên mua (`ENTRY-BUY`), mua thứ cấp thỏa thuận (`TRADE-BUY`), hoặc bán thứ cấp thỏa thuận (`TRADE-SELL`). Tính toán đơn giá, số lượng đặt tối thiểu/tối đa, các bậc phí giao dịch, thuế tiêu thụ và số dư tiền khả dụng hoặc số lượng ST có thể bán.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `GET` / `POST`
- **URL**: `/order/buy/input` (Rewrite nội bộ: `/index.php?c=order&a=buyInput`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Chi tiết Request (Request Details)
- **Parameters / Body**:
  - `FUND_CD`: `string` (Bắt buộc)
  - `TRANSACTION_TYPE`: `string` (Tùy chọn, mặc định là `BUY`; các giá trị: `BUY`, `ENTRY-BUY`, `TRADE-BUY`, `TRADE-SELL`)

#### Chi tiết Response (Response Details)
- **Response Schema**:
  ```json
  {
    "STATUS": "OK",
    "ERROR": null,
    "DATA": {
      "FUND": {
        "FUND_CD": "string",
        "FUND_NM": "string",
        "PRICE": "string (number)",
        "MIN_QTY": "string",
        "UNIT_QTY": "string",
        "MAX_QTY": "string",
        "BUYABLE_AMOUNT": "string"
      },
      "COMMISSION": {
        "RATE": "string",
        "TAX_RATE": "string"
      },
      "CHECK_IF_TRANS_INSIDER": "integer (0|1)"
    }
  }
  ```
- **Mã lỗi nghiệp vụ (Business Error Codes)**: `fund_not_exist`, `buy_order_un_right_time`, `entry_order_period_error`, `entry_not_elected`, `entry_buy_already_purchased`.

---

### 15. Xác nhận lệnh (Order Confirmation)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Order Confirmation (`ST-API-ORD-007`)
- **Controller - Service**: `App\Controllers\OrderController::BuyConfirm` -> `BuyService::buyConfirm` / `TradeService::tradeBuyConfirm` / `TradeService::tradeSellConfirm`
- **Tệp nguồn / Class**: [`App/Controllers/OrderController.php::BuyConfirm`](/st-api/App/Controllers/OrderController.php#L106-L143)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Kiểm tra tính hợp lệ toàn diện của lệnh đặt (số lượng đặt, bội số đơn vị, hạn mức còn lại của quỹ, hạn mức tối đa mỗi ngày của nhà đầu tư, tiền/chứng khoán khả dụng, và thỏa thuận bảo mật người nội bộ). Lưu tạm preview lệnh vào Redis (`TMP_ORDER:<order_id>`) phục vụ cho bước thực thi sau đó.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `POST`
- **URL**: `/order/buy/confirm` (Rewrite nội bộ: `/index.php?c=order&a=buyConfirm`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Xác thực & Phân quyền (Authentication & Authorization)
- **Loại xác thực (Auth Type)**: Bearer Token + Token chống giả mạo `X-Ct` (Bắt buộc trong các môi trường khác local; xác thực qua key Redis `ct:<token>`).

#### Chi tiết Request (Request Details)
- **Headers**: Các header chuẩn của `AppController` + `X-Ct: <token>`.
- **Body Schema**:
  ```json
  {
    "TRANSACTION_TYPE": "string (Bắt buộc: BUY, ENTRY-BUY, TRADE-BUY, hoặc TRADE-SELL)",
    "FUND_CD": "string (Bắt buộc)",
    "ORDER_QTY": "string (Bắt buộc, integer string)",
    "CONFIRM_INSIDER_CHECK": "integer (tùy chọn, bắt buộc nếu người dùng thuộc danh sách nội bộ)"
  }
  ```

#### Chi tiết Response (Response Details)
- **Response Schema**: Trả về đầy đủ các số liệu đã tính toán bao gồm `ORDER_AMOUNT`, `COMMISSION_AMOUNT`, `TAX_AMOUNT`, `TOTAL_PAYMENT_AMOUNT` và key lưu tạm đơn hàng.
- **Mã lỗi nghiệp vụ (Business Error Codes)**: `buy_order_input`, `buy_order_min_qty`, `buy_order_unit_qty`, `buy_order_max_qty`, `buy_order_buyable_cash`, `buy_fund_max_qty_shortage`, `buy_order_un_confirmed_doc`, `fund_user_in_stock_error`.

#### Tác dụng phụ (Side Effects)
- **Caching**: Lưu bản ghi xem trước lệnh vào Redis `TMP_ORDER:<order_id>` với TTL ngắn.
- **Hủy token**: Tiêu thụ và xóa token `X-Ct` (`ct:<token>`) khỏi Redis ngay sau khi hoàn thành bước này.

---

### 16. Hoàn tất đặt lệnh (Order Execution Complete)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Order Execution Complete (`ST-API-ORD-008`)
- **Controller - Service**: `App\Controllers\OrderController::BuyComplete` -> `BuyService::buyComplete` / `TradeService::tradeBuyComplete` / `TradeService::tradeSellComplete`
- **Tệp nguồn / Class**: [`App/Controllers/OrderController.php::BuyComplete`](/st-api/App/Controllers/OrderController.php#L145-L185)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Thực thi chính thức và cam kết lệnh tài chính vào hệ thống. Xác thực mã PIN/mật khẩu giao dịch, cập nhật số dư tiền hoặc số lượng chứng khoán của nhà đầu tư qua engine sổ cái (`netsec/common-balance`), tạo bản ghi lệnh (`PST_BUY_ORDER` hoặc `PST_PROP_TRADE_ORDER`), cập nhật số lượng phân bổ tích lũy, và đưa thông báo giao dịch vào hàng đợi.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `POST`
- **URL**: `/order/buy/complete` (Rewrite nội bộ: `/index.php?c=order&a=buyComplete`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Xác thực & Phân quyền (Authentication & Authorization)
- **Loại xác thực (Auth Type)**: Bearer Token + Token chống giả mạo `X-Ct`.

#### Chi tiết Request (Request Details)
- **Body Schema**:
  ```json
  {
    "TRANSACTION_TYPE": "string (Bắt buộc: BUY, ENTRY-BUY, TRADE-BUY, hoặc TRADE-SELL)",
    "FUND_CD": "string (Bắt buộc)",
    "ORDER_QTY": "string (Bắt buộc)",
    "PIN": "string (Bắt buộc: mã PIN giao dịch 4-8 chữ số)"
  }
  ```

#### Chi tiết Response (Response Details)
- **Schema**:
  ```json
  {
    "STATUS": "OK",
    "ERROR": null,
    "DATA": {
      "ORDER_NUM": "string",
      "FUND_CD": "string",
      "ORDER_QTY": "string",
      "ORDER_AMOUNT": "string"
    }
  }
  ```
- **Mã lỗi nghiệp vụ (Business Error Codes)**: `pin_error`, `pass_locked`, `E005-0013`, `E005-0016`, `buy_order_buyable_cash`, `fund_in_stock_error`, `self_cash_balance_update_failed`.

#### Luồng xử lý & Giao dịch (Processing Flow & Transactions)
- **Luồng xử lý**:
  1. Kiểm tra tính hợp lệ của token `X-Ct`.
  2. Xác minh mã PIN giao dịch với `ECO_USER` (theo dõi số lần nhập sai liên tiếp; khóa tài khoản khi vượt quá giới hạn bằng mã `E005-0013`).
  3. Khởi tạo transaction database nguyên tử (`AppModel::beginTransaction()`).
  4. Gọi `CashAbleModel` / thư viện sổ cái `common-balance` để đóng băng/chuyển dịch số dư tiền.
  5. Chèn bản ghi lệnh vào `PST_BUY_ORDER` với trạng thái `0` (chờ quyết toán).
  6. Cập nhật bộ đếm tích lũy trong `PST_FUND_APPRICATION`.
  7. Chèn thông báo giao dịch vào queue `PST_TRADE_NOTIFICATION`.
  8. Commit transaction database (`AppModel::commitTransaction()`).
  9. Xóa cache xem trước trong Redis và xóa token `X-Ct`.
- **Ranh giới giao dịch**: Transaction ACID toàn vẹn bao trọn các thao tác trừ tiền, ghi nhận lệnh và hàng đợi gửi thông báo.

---

### 17. Hủy lệnh (Order Cancellation)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Order Cancellation (`ST-API-ORD-009`)
- **Controller - Service**: `App\Controllers\OrderController::Cancel` -> `BuyService::buyCancel` / `TradeService::buyCancel` / `TradeService::sellCancel`
- **Tệp nguồn / Class**: [`App/Controllers/OrderController.php::Cancel`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/OrderController.php#L187-L224)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Hủy một lệnh đang hoạt động chưa khớp (lệnh mua phát hành sơ cấp, lệnh mua thứ cấp, hoặc lệnh bán thứ cấp). Giải phóng số tiền đã đóng băng trở lại số dư khả dụng của nhà đầu tư hoặc khôi phục lại số lượng ST khả dụng.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `POST`
- **URL**: `/order/cancel` (Rewrite nội bộ: `/index.php?c=order&a=cancel`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Xác thực & Phân quyền (Authentication & Authorization)
- **Loại xác thực (Auth Type)**: Bearer Token + Token chống giả mạo `X-Ct`.

#### Chi tiết Request (Request Details)
- **Body Schema**:
  ```json
  {
    "CANCEL_TYPE": "string (Bắt buộc: BUY, TRADE-BUY, hoặc TRADE-SELL)",
    "ORDER_NUM": "string (Bắt buộc: mã định danh chuỗi lệnh)"
  }
  ```

#### Chi tiết Response (Response Details)
- **Schema**:
  ```json
  {
    "STATUS": "OK",
    "ERROR": null,
    "DATA": {
      "ORDER_NUM": "string",
      "ORDER_STATUS": 10
    }
  }
  ```
- **Mã lỗi nghiệp vụ (Business Error Codes)**: `order_not_exist`, `order_cancel_invalid`, `buy_order_un_right_time`.

#### Luồng xử lý & Giao dịch (Processing Flow & Transactions)
- **Luồng xử lý**:
  1. Kiểm tra trạng thái lệnh phải là chưa khớp (`ORDER_STATUS = 0`).
  2. Xác minh yêu cầu hủy được thực hiện trước thời hạn cut-off trong ngày (`pfund.order.deadline`).
  3. Bắt đầu transaction database.
  4. Cập nhật `ORDER_STATUS = 10` (Đã hủy).
  5. Mở đóng băng tiền qua `CashAbleModel` hoặc khôi phục vị thế ST trong `FundRealBalanceModel`.
  6. Commit transaction database.
  7. Xóa token `X-Ct`.

---

### 18. Xác nhận người nội bộ quỹ (Fund Insider Confirmation)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Fund Insider Confirmation (`ST-API-ORD-010`)
- **Controller - Service**: `App\Controllers\OrderController::insiderconfirm` -> `App\Service\InsiderConfirmService::insiderConfirm`
- **Tệp nguồn / Class**: [`App/Controllers/OrderController.php::insiderconfirm`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/OrderController.php#L229-L237)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Kiểm tra tính tuân thủ pháp lý nhằm xác định xem nhà đầu tư đang đăng nhập có phải là người nội bộ doanh nghiệp (役員・主要株主 - Thành viên ban điều hành / Cổ đông lớn) của đơn vị phát hành hoặc các công ty liên kết với quỹ mục tiêu hay không.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `GET` / `POST`
- **URL**: `/order/insiderconfirm` (Rewrite nội bộ: `/index.php?c=order&a=insiderconfirm`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Chi tiết Request & Response (Request & Response Details)
- **Parameters**: `FUND_CD`: `string` (Bắt buộc)
- **Response Schema**:
  ```json
  {
    "STATUS": "OK",
    "ERROR": null,
    "DATA": {
      "FUND_CD": "string",
      "INSIDER_CONFIRMED_FLG": "integer (0: unconfirmed, 1: confirmed)",
      "FUND_INSIDER_FLG": "integer (0: not insider, 1: is insider)",
      "COMPANY_NAME_LIST": ["string"],
      "FUND_INSIDER_ARR": [
        {
          "FUND_CD": "string",
          "BRAND_CD": "string",
          "BRAND_NAME": "string",
          "INSIDER_TYPE": "string",
          "FAMILY_NAME": "string"
        }
      ]
    }
  }
  ```
- **Dependencies**: `ECO_USER_INSIDER`, `PST_FUND_INSIDER_COMPANY`.

---

### 19. Trang MyPage đăng ký ưu tiên (Priority Entry MyPage)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Priority Entry MyPage (`ST-API-PRI-001`)
- **Controller - Service**: `App\Controllers\PrientryController::mypage` -> `App\Service\PrientryService::fundListForPreSale`
- **Tệp nguồn / Class**: [`App/Controllers/PrientryController.php::mypage`](/st-api/App/Controllers/PrientryController.php#L17-L24)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Truy xuất danh sách các quỹ đang trong giai đoạn đăng ký tham gia bốc thăm quyền mua ưu tiên (優先申込エントリー - *Yusen Moshikomi Entry*) và các quỹ đang trong giai đoạn quay số bốc thăm (抽選期間中).

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `GET`
- **URL**: `/prientry/mypage` (Rewrite nội bộ: `/index.php?c=prientry&a=mypage`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Chi tiết Response (Response Details)
- **Schema**:
  ```json
  {
    "STATUS": "OK",
    "ERROR": null,
    "DATA": {
      "FUND_IN_PRI_ENTRY": [
        {
          "FUND_CD": "string",
          "FUND_NM": "string",
          "PRI_ENTRY_START_DT": "string",
          "PRI_ENTRY_END_DT": "string",
          "PRI_ENTRY": {
            "ENTRY_QTY": "string",
            "ENTRY_AMOUNT": "string"
          }
        }
      ],
      "FUND_IN_LOTTERY": ["array"]
    }
  }
  ```
- **Dependencies**: `PST_FUND`, `PST_PRI_ENTRY`, `PST_FUND_TAG`.

---

### 20. Chi tiết quỹ đăng ký ưu tiên (Priority Entry Fund Detail)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Priority Entry Fund Detail (`ST-API-PRI-002`)
- **Controller - Service**: `App\Controllers\PrientryController::fund` -> `FundService::detail` & `PrientryService::getEntry`
- **Tệp nguồn / Class**: [`App/Controllers/PrientryController.php::fund`](/st-api/App/Controllers/PrientryController.php#L31-L44)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Cung cấp chi tiết các thông số kỹ thuật của quỹ đang trong giai đoạn bốc thăm ưu tiên, kèm theo trạng thái đơn đăng ký của người dùng nếu đã đăng ký trước đó.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `GET`
- **URL**: `/prientry/fund/{FUND_CD}` (Rewrite nội bộ: `/index.php?c=prientry&a=fund&FUND_CD=$1`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Chi tiết Request (Request Details)
- **Route / Query Parameter**: `FUND_CD`: `string` (Bắt buộc)

#### Chi tiết Response (Response Details)
- **Schema**:
  ```json
  {
    "STATUS": "OK",
    "ERROR": null,
    "DATA": {
      "FUND": "object",
      "PRI_ENTRY": {
        "ENTRY_STATUS": "integer",
        "ORDER_QTY": "string",
        "ORDER_AMOUNT": "string"
      }
    }
  }
  ```

---

### 21. Đăng ký tham gia bốc thăm ưu tiên (Priority Entry Application)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Priority Entry Application (`ST-API-PRI-003`)
- **Controller - Service**: `App\Controllers\PrientryController::entry` -> `App\Service\PrientryService::entryComplete`
- **Tệp nguồn / Class**: [`App/Controllers/PrientryController.php::entry`](/st-api/App/Controllers/PrientryController.php#L51-L67)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Gửi đơn đăng ký tham gia bốc thăm để giành quyền mua ưu tiên trước khi quỹ mở đợt phát hành rộng rãi ra công chúng. Kiểm tra thời hạn đăng ký, số suất đăng ký tối thiểu/tối đa và hiệu lực của thẻ lưu trú (thẻ cư trú).

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `POST`
- **URL**: `/prientry/entry/{FUND_CD}` (Rewrite nội bộ: `/index.php?c=prientry&a=entry&FUND_CD=$1`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Xác thực & Phân quyền (Authentication & Authorization)
- **Loại xác thực (Auth Type)**: Bearer Token + Token chống giả mạo `X-Ct`.

#### Chi tiết Request (Request Details)
- **Route Parameter**: `FUND_CD`: `string` (Bắt buộc)
- **Body Schema**:
  ```json
  {
    "ORDER_QTY": "string (Bắt buộc: số lượng suất đăng ký)"
  }
  ```

#### Chi tiết Response (Response Details)
- **Schema**:
  ```json
  {
    "STATUS": "OK",
    "ERROR": null,
    "DATA": {
      "FUND_CD": "string",
      "FUND_NM": "string",
      "ICON": "string"
    }
  }
  ```
- **Mã lỗi nghiệp vụ (Business Error Codes)**: `entry_input_qty`, `entry_min_qty`, `entry_bought`, `entry_fund_not_start`, `entry_fund_end`, `entry_max_qty_fund`, `entry_user_status_stop_buy_flg`.

#### Luồng xử lý & Giao dịch (Processing Flow & Transactions)
- **Luồng xử lý**: Khởi tạo database transaction, tạo bản ghi đăng ký trong `PST_PRI_ENTRY`, cập nhật `PST_PRI_ENTRY_SUM`, commit.
- **Tác dụng phụ**: Tiêu thụ và xóa token `X-Ct`.

---

### 22. Cập nhật số lượng đăng ký ưu tiên (Priority Entry Update Quantity)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Priority Entry Update Quantity (`ST-API-PRI-004`)
- **Controller - Service**: `App\Controllers\PrientryController::update` -> `App\Service\PrientryService::updateComplete`
- **Tệp nguồn / Class**: [`App/Controllers/PrientryController.php::update`](/st-api/App/Controllers/PrientryController.php#L74-L90)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Điều chỉnh số lượng suất đăng ký tham gia bốc thăm trong khi thời gian mở đăng ký vẫn còn hiệu lực.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `POST`
- **URL**: `/prientry/update/{FUND_CD}` (Rewrite nội bộ: `/index.php?c=prientry&a=update&FUND_CD=$1`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Xác thực & Phân quyền (Authentication & Authorization)
- **Loại xác thực (Auth Type)**: Bearer Token + Token chống giả mạo `X-Ct`.

#### Chi tiết Request (Request Details)
- **Route Parameter**: `FUND_CD`: `string` (Bắt buộc)
- **Body Schema**:
  ```json
  {
    "ORDER_QTY": "string (Bắt buộc: số lượng suất đăng ký mới)"
  }
  ```

#### Chi tiết Response (Response Details)
- **Schema**: Trả về metadata của quỹ đã được cập nhật (`FUND_CD`, `FUND_NM`, `ICON`).
- **Mã lỗi nghiệp vụ (Business Error Codes)**: `entry_not_applied`, `entry_not_update`, `entry_fund_end`.

---

### 23. Hủy đăng ký ưu tiên (Priority Entry Cancellation)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: Priority Entry Cancellation (`ST-API-PRI-005`)
- **Controller - Service**: `App\Controllers\PrientryController::cancel` -> `App\Service\PrientryService::cancelComplete`
- **Tệp nguồn / Class**: [`App/Controllers/PrientryController.php::cancel`](st-api/App/Controllers/PrientryController.php#L97-L113)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Hủy bỏ đơn đăng ký tham gia bốc thăm quyền mua ưu tiên trước khi thời gian đăng ký kết thúc.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `POST`
- **URL**: `/prientry/cancel/{FUND_CD}` (Rewrite nội bộ: `/index.php?c=prientry&a=cancel&FUND_CD=$1`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Xác thực & Phân quyền (Authentication & Authorization)
- **Loại xác thực (Auth Type)**: Bearer Token + Token chống giả mạo `X-Ct`.

#### Chi tiết Request (Request Details)
- **Route Parameter**: `FUND_CD`: `string` (Bắt buộc)

#### Chi tiết Response (Response Details)
- **Schema**: Trả về metadata quỹ vừa hủy đơn (`FUND_CD`, `FUND_NM`, `ICON`).
- **Mã lỗi nghiệp vụ (Business Error Codes)**: `entry_not_applied`, `entry_fund_end`.

---

### 24. Webhook thông báo gửi email SendGrid (SendGrid Webhook Notification)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: SendGrid Webhook Notification (`ST-API-NOT-001`)
- **Controller - Service**: `App\Controllers\NotificationController::sendgridWebhook` -> `App\Service\WebhookService::sendGrid`
- **Tệp nguồn / Class**: [`App/Controllers/NotificationController.php::sendgridWebhook`](/st-api/App/Controllers/NotificationController.php#L13-L37)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Tiếp nhận bất đồng bộ các webhook trạng thái gửi email từ SendGrid cho các email thông báo giao dịch (xác nhận đơn hàng, chứng từ khớp lệnh, thông báo giao kết hợp đồng), thực hiện kiểm tra chữ ký mã hóa ECDSA và cập nhật trạng thái gửi thư trong `PST_TRADE_NOTIFICATION`.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `POST`
- **URL**: `/webhook/transfer-notification/sendgrid/{x-ci}` (Rewrite nội bộ: `/index.php?c=notification&a=sendgridWebhook&x-ci=$1`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Xác thực & Phân quyền (Authentication & Authorization)
- **Loại xác thực (Auth Type)**: Xác thực chữ ký Webhook ECDSA (`SendGrid\EventWebhook\EventWebhook`).
- **Quyền hạn (Permissions)**: Không yêu cầu (Webhook công khai). Kế thừa trực tiếp `BraveController`, bỏ qua các bước kiểm tra session người dùng của `AppController`.
- **Headers đặc thù**:
  - `X-Twilio-Email-Event-Webhook-Signature`: Chuỗi chữ ký ECDSA (Bắt buộc)
  - `X-Twilio-Email-Event-Webhook-Timestamp`: Timestamp của event (Bắt buộc)

#### Chi tiết Request (Request Details)
- **Route Parameter**: `x-ci`: Mã định danh công ty (xác định context database của tenant)
- **Body Schema**: Mảng JSON chứa các SendGrid Event objects:
  ```json
  [
    {
      "email": "investor@example.com",
      "event": "delivered",
      "nid": "10042",
      "timestamp": 1757400000
    }
  ]
  ```

#### Chi tiết Response (Response Details)
- **Status Codes**: `200 OK` khi thành công, `401 Unauthorized` khi chữ ký không hợp lệ.
- **Mã lỗi nghiệp vụ (Business Error Codes)**: `sendgrid_verification_error`, `unknown_error`.

#### Luồng xử lý & Giao dịch (Processing Flow & Transactions)
- **Luồng xử lý**:
  1. Đọc khóa `transfer_notification.sendgrid_public_verification_key` từ cấu hình.
  2. Xác minh chữ ký ECDSA đối chiếu với raw HTTP payload.
  3. Bắt đầu transaction database.
  4. Cập nhật trạng thái thư trong `PST_TRADE_NOTIFICATION` (`MAIL_STATUS`: 1 = processed, 2 = dropped/bounce, 10 = delivered, 11 = open/click) và ghi nhận timestamp xử lý.
  5. Commit transaction database.

---

### 25. Proxy hình ảnh icon S3 (S3 Icon Image Proxy)

#### Định danh & Quản trị (Identification & Management)
- **Tên API / ID**: S3 Icon Image Proxy (`ST-API-MED-001`)
- **Controller - Service**: `App\Controllers\UnauthCommonController::icon`
- **Tệp nguồn / Class**: [`App/Controllers/UnauthCommonController.php::icon`](/st-api/App/Controllers/UnauthCommonController.php#L13-L57)
- **Trạng thái**: Implemented / Active
- **Ngày / Phiên bản**: Current Master / v1.0

#### Mô tả nghiệp vụ (Business Description)
Truy xuất các tài nguyên thương hiệu, huy hiệu và icon công khai của quỹ được lưu trữ trên AWS S3 dưới dạng Data URI mã hóa Base64 phục vụ cho ứng dụng web và mobile.

#### Endpoint & Môi trường (Endpoint & Environment)
- **Method**: `GET`
- **URL**: `/s3/{x-ci}/{icon-key}` (Rewrite nội bộ: `/index.php?c=unauthCommon&a=icon&x-ci=$1&icon-key=$2`)
- **Môi trường**: Tất cả môi trường
- **Phiên bản**: v1

#### Xác thực & Phân quyền (Authentication & Authorization)
- **Loại xác thực (Auth Type)**: Bearer Token (kế thừa kiểm tra session của `AppController`).
- **Kiểm tra an toàn (Security Check)**: `icon-key` bắt buộc phải bắt đầu bằng tiền tố `public` và có đuôi tệp ảnh hợp lệ (`png`, `jpeg`, `jpg`, `gif`, `svg`, v.v.).

#### Chi tiết Request (Request Details)
- **Parameters**:
  - `x-ci`: Mã công ty (Bắt buộc)
  - `icon-key`: Đường dẫn key của object trên S3 (Bắt buộc, ví dụ: `public/funds/fnd01_icon.png`)

#### Chi tiết Response (Response Details)
- **Schema**:
  ```json
  {
    "STATUS": "OK",
    "ERROR": "",
    "DATA": {
      "ICON": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA..."
    },
    "now": "2026-09-09 10:20:00",
    "APP_ENV": "local"
  }
  ```
- **Mã lỗi nghiệp vụ (Business Error Codes)**: `unknown_error`

---

## 4. Ghi chú & Lưu ý Kiến trúc Quan trọng (Notes & Important Architectural Caveats)

1. **Sự khác biệt trong định tuyến Nginx (Đặc biệt quan trọng cho thành viên mới)**:
   - Trong tệp cấu hình `local-docker/etc/local/nginx/pfund-api.conf`, có các quy tắc rewrite cho `/order/sell/input`, `/order/sell/confirm`, và `/order/sell/complate`. Tuy nhiên, trong `OrderController.php` **KHÔNG** hề có các phương thức `sellInput`, `sellConfirm` hay `sellComplate`.
   - **Cách sử dụng đúng**: Mọi thao tác bán trên thị trường thứ cấp bắt buộc phải được gọi thông qua các endpoint `/order/buy/input`, `/order/buy/confirm`, và `/order/buy/complete` với tham số `TRANSACTION_TYPE: 'TRADE-SELL'`. Nếu gọi trực tiếp đến các đường dẫn `/order/sell/*`, dispatcher sẽ báo lỗi `404 router error`.
   - Tương tự, các đường dẫn `/history/trade/detail`, `/history/order/detail`, `/history/dividend/detail`, và `/history/refund/detail` trong cấu hình Nginx là các rewrite stubs chưa được triển khai mã nguồn xử lý. Toàn bộ việc truy vấn lịch sử được thực hiện qua các API danh sách tương ứng (`/history/trade/list`, `/history/order/list`, v.v.).

2. **Quy tắc bắt buộc về số học dấu phẩy tĩnh (`ext-bcmath`)**:
   - Là một dịch vụ giao dịch tài chính chứng khoán, việc sử dụng các kiểu dữ liệu số thực dấu phẩy động nguyên bản của PHP (`float`, `double`) bị nghiêm cấm hoàn toàn trong bất kỳ phép tính nào liên quan đến khối lượng lệnh, đơn giá, phí giao dịch, cổ tức hay thuế.
   - Tất cả các phép toán phải sử dụng chuỗi và các hàm BCMath (`bcadd`, `bcsub`, `bcmul`, `bcdiv`, `bccomp`) với scale được chỉ định tường minh.

3. **Multi-Tenancy & Định tuyến Cơ sở dữ liệu (`X-Ci`)**:
   - Mỗi request đều yêu cầu header `X-Ci`. Hệ thống truy vấn `basis.BCO_COMPANY` với điều kiện `COMPANY_ID = :X-Ci` để xác định tiền tố cơ sở dữ liệu. Nếu bảng `BCO_COMPANY` chưa có dữ liệu của tenant tương ứng, API sẽ gặp lỗi kết nối `Unknown database '_pfund'`.

4. **Xác thực hai bước & Token chống giả mạo (`X-Ct`)**:
   - Các tác vụ ghi quan trọng (`/order/buy/confirm`, `/order/buy/complete`, `/order/cancel`, `/prientry/entry/*`, `/prientry/update/*`, `/prientry/cancel/*`) bắt buộc phải truyền header `X-Ct` trên các môi trường khác local.
   - Frontend sẽ lấy hoặc tạo token này trong Redis (`ct:<token>`). Token sẽ được kiểm tra và lập tức xóa bỏ (`deleteXct()`) để ngăn chặn tấn công replay và việc gửi trùng lặp yêu cầu (double submission).

5. **Kiến trúc Session & Xác thực Token không phụ thuộc DB**:
   - `st-api` không thực hiện truy vấn cơ sở dữ liệu SQL để xác thực Bearer token ở mỗi HTTP request. Hệ thống truy vấn trực tiếp vào Redis (`TK:<COMPANY_SEQ_NO>:<token>`), đạt tốc độ xác thực tức thì O(1) dưới 1 miligiây mà không gây quá tải cho hệ quản trị cơ sở dữ liệu quan hệ.
