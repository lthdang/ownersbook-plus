# API Specification: st-api (Fund Core API / Security Token Trading API)

## 1. Project Information

- **Project Name**: `st-api` (`ST-FRONT-API`)
- **Primary Language**: PHP 8.3 (tested in runtime environment PHP 8.3.16 on container `st-app-pfund`)
- **Framework & Software Architecture**:
  - **Framework**: Custom lightweight Brave Framework (MVC-like structure with `BraveDispatcher`, `BraveController`, `AppController`).
  - **Architecture**: Layered architecture comprising HTTP Dispatcher -> Request Interceptor / Middlewares (`AppController`) -> Controller (`App\Controllers`) -> Service Layer (`App\Service`) -> Model Layer / Database Abstraction (`App\Models`, `BraveModel`).
  - **Ledger & Accounting Engine**: `netsec/common-balance` (`CashAbleModel`, `CashRealBalanceModel`) utilizing arbitrary-precision decimal operations via PHP `ext-bcmath` (strict prohibition of native floating-point calculations for financial integrity).
  - **Session & Caching Layer**: Redis (OAuth2 Bearer token sessions `TK:<company_seq>:<access_token>`, rate limiting counters `RATE_LIMIT:<user_seq>:<window>`, transaction anti-tamper tokens `ct:<token>`, order preview caching `TMP_ORDER:<order_id>`).
  - **Reverse Proxy / Routing**: Nginx reverse proxy rewrite engine (`pfund-api.conf`) mapping RESTful paths to Brave query parameters `/index.php?c=<controller>&a=<action>`.
  - **Telemetry & Monitoring**: Sentry SDK (`sentry/sdk: ^3.1`), Monolog (`monolog/monolog: ^2.3`), Fluentd logger (`fluent/logger: ^1.0`).
- **Database & Data Access Layer**:
  - **Abstraction**: `adodb/adodb-php: 5.22.7` integrated via `BraveMySQLBuilder`.
  - **Multi-Tenancy Resolution**: Dynamic database routing determined by incoming request header `X-Ci` (Company ID). System looks up `basis.BCO_COMPANY` where `COMPANY_ID = :x_ci` to obtain `COMPANY_SEQ_NO` and `DB_NAME` (`{prefix}_pfund`), dynamically switching the active database connection.
  - **Database Cluster**:
    - `basis`: Shared system database holding `BSYS_CONFIG`, `BSYS_MAINTENANCE`, `BSYS_MAINTENANCE_PERIODIC`, `BCO_COMPANY`.
    - `{prefix}_pfund`: Fund & Security Token trading database holding `PST_FUND`, `PST_BUY_ORDER`, `PST_PRICE`, `PST_CONTRACT_PDF`, `PST_CONTRACT_LOG`, `PST_PRI_ENTRY`, `PST_INCOME_TAX`, etc.
    - `{prefix}`: Customer / company-specific core database holding `ECO_USER`, `ECO_DEVICE`, `ECO_TRADE_HIST`, `ECO_BASE_DATE`, `ECO_ELECTRONIC_DELIVERY`, `ECO_USER_INSIDER`.

---

## 2. Summary Table (Index)

| API Name | Method | URL | Module | Status |
|---|---|---|---|---|
| [Asset Summary](#1-asset-summary) | GET | `/asset/summary` | Asset / Portfolio | Implemented |
| [Fund List](#2-fund-list) | GET | `/fund/list` | Fund | Implemented |
| [Fund Detail](#3-fund-detail) | GET | `/fund/detail` | Fund | Implemented |
| [Order History List](#4-order-history-list) | POST | `/history/order/list` | History | Implemented |
| [Trade Execution History List](#5-trade-execution-history-list) | POST | `/history/trade/list` | History | Implemented |
| [Dividend & Refund History List](#6-dividend--refund-history-list) | POST | `/history/dividend_refund/list` | History | Implemented |
| [Settlement History List](#7-settlement-history-list) | POST | `/history/all/list` | History | Implemented |
| [Income Tax Support Data](#8-income-tax-support-data) | GET / POST | `/history/incometax` | History / Tax | Implemented |
| [Contract Document List](#9-contract-document-list) | GET / POST | `/order/contract/list` | Order / Contract | Implemented |
| [Contract Document Read](#10-contract-document-read) | GET / POST | `/order/contract/read` | Order / Contract | Implemented |
| [Contract Document Accept](#11-contract-document-accept) | POST | `/order/contract/accept` | Order / Contract | Implemented |
| [Text Document List](#12-text-document-list) | GET | `/order/textdocument/list` | Order / Text Document | Implemented |
| [Text Document Accept](#13-text-document-accept) | POST | `/order/textdocument/accept` | Order / Text Document | Implemented |
| [Order Input Information](#14-order-input-information) | GET / POST | `/order/buy/input` | Order / Trading | Implemented |
| [Order Confirmation](#15-order-confirmation) | POST | `/order/buy/confirm` | Order / Trading | Implemented |
| [Order Execution Complete](#16-order-execution-complete) | POST | `/order/buy/complete` | Order / Trading | Implemented |
| [Order Cancellation](#17-order-cancellation) | POST | `/order/cancel` | Order / Trading | Implemented |
| [Fund Insider Confirmation](#18-fund-insider-confirmation) | GET / POST | `/order/insiderconfirm` | Order / Compliance | Implemented |
| [Priority Entry MyPage](#19-priority-entry-mypage) | GET | `/prientry/mypage` | Priority Entry | Implemented |
| [Priority Entry Fund Detail](#20-priority-entry-fund-detail) | GET | `/prientry/fund/{FUND_CD}` | Priority Entry | Implemented |
| [Priority Entry Application](#21-priority-entry-application) | POST | `/prientry/entry/{FUND_CD}` | Priority Entry | Implemented |
| [Priority Entry Update Quantity](#22-priority-entry-update-quantity) | POST | `/prientry/update/{FUND_CD}` | Priority Entry | Implemented |
| [Priority Entry Cancellation](#23-priority-entry-cancellation) | POST | `/prientry/cancel/{FUND_CD}` | Priority Entry | Implemented |
| [SendGrid Webhook Notification](#24-sendgrid-webhook-notification) | POST | `/webhook/transfer-notification/sendgrid/{x-ci}` | Notification / Webhook | Implemented |
| [S3 Icon Image Proxy](#25-s3-icon-image-proxy) | GET | `/s3/{x-ci}/{icon-key}` | Media / Proxy | Implemented |

---

## 3. Main Content: API Details

---

### 1. Asset Summary

#### Identification & Management
- **API Name / ID**: Asset Summary (`ST-API-AST-001`)
- **Controller - Service**: `App\Controllers\AssetController::summary` -> `App\Service\FundBalanceService::summary`
- **Source File / Class**: [`App/Controllers/AssetController.php::summary`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/AssetController.php#L10-L14)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Retrieves an aggregate snapshot of the logged-in investor's asset portfolio, including cash deposits (預り金 - *Azukarikin*), available cash for purchasing (買付可能額 - *Kaitsuke Kanogaku*), total valuation of held Security Tokens (ST評価額合計), unsettled buy trade valuations, and fund-by-fund asset holdings breakdown.

#### Endpoint & Environment
- **Method**: `GET`
- **URL**: `/asset/summary` (Internal rewrite: `/index.php?c=asset&a=summary`)
- **Environment**: All environments (Local, Dev, Staging, Production)
- **Version**: v1

#### Authentication & Authorization
- **Auth Type**: Bearer Token (OAuth2 access token issued by `passport` service)
- **Permissions**: Verified client session in Redis (`TK:<COMPANY_SEQ_NO>:<token>`)
- **Special Headers**:
  - `Authorization`: `Bearer <token>` (Required)
  - `X-Ci`: Company ID (e.g., `sgDj8sHr`) (Required)
  - `X-Pf`: Client platform (`Web`, `iOS`, `Android`) (Required)
  - `X-Sv`: Service sequence number (Required)
  - `X-Du`: Client device UUID matching `ECO_DEVICE.CREDIBLE_FLG = '1'` (Required)
- **Rate Limits**: Redis sliding counter `RATE_LIMIT:<USER_SEQ_NO>:<window>` (Default: max 30 requests per 60 seconds).

#### Request Details
- **Headers**:
  ```http
  Authorization: Bearer <access_token>
  X-Ci: sgDj8sHr
  X-Pf: Web
  X-Sv: 1
  X-Du: 12345678-1234-4234-8234-1234567890ab
  ```
- **Query Parameters**: None (or optional `TYPE=ASSETS` / `TYPE=BALANCE` used by frontend filters)
- **Body Schema**: None
- **Sample Payload**: None

#### Response Details
- **Status Codes**: `200 OK` (HTTP status code is 200 even for logical errors, wrapped with `STATUS: "OK"` or `"NG"`)
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
- **Business Error Codes**: `unauth` (401), `xdu_credible_error`, `rate_limit_msg`, `no_account_number_verified_error`
- **Sample Response**:
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

#### Processing Flow & Transactions
- **Flow**:
  1. Base controller verifies maintenance window, Bearer token, rate limit, device credibility, and corporate account status.
  2. Queries `BaseDateModel` for current trading base date (`CURRENT_D`).
  3. Queries `CashAbleModel` (`netsec/common-balance`) for deposit and buyable amounts.
  4. Queries `FundRealBalanceModel` for owned funds and calculates ST evaluation using `PriceModel` prices via `AssetHelper::fixForSettlement`.
  5. Computes total portfolio evaluation using `ext-bcmath`.
- **Transaction Boundaries**: Read-only, no database transaction started.
- **Idempotency**: Idempotent (`GET`).

#### Side Effects
- **Caching**: Updates Redis token expiration (`TK:...` extended by 3600s) and increments rate limit counter.
- **Logging**: Access logged via Monolog/Fluentd.

#### Dependencies
- **Internal Services**: `FundBalanceService`, `CashAbleModel` (`netsec/common-balance`), `PriceModel`, `TradeHistModel`.
- **External Services**: Redis, MySQL (`basis`, `{prefix}`, `{prefix}_pfund`).
- **Related APIs**: `/fund/list`, `/order/buy/input`.

---

### 2. Fund List

#### Identification & Management
- **API Name / ID**: Fund List (`ST-API-FND-001`)
- **Controller - Service**: `App\Controllers\FundController::List` -> `App\Service\FundService::fundListForFind` / `FundModel::fundListSmall`
- **Source File / Class**: [`App/Controllers/FundController.php::List`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/FundController.php#L13-L58)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Fetches a list of funds according to filtering and search criteria. In the current implementation, strict validation requires `TYPE=simple` (compact fund list for dropdowns) or `TYPE=FIND` (full search with filters like fund status, yield range, tags).

#### Endpoint & Environment
- **Method**: `GET`
- **URL**: `/fund/list` (Internal rewrite: `/index.php?c=fund&a=list`)
- **Environment**: All environments
- **Version**: v1

#### Authentication & Authorization
- **Auth Type**: Bearer Token
- **Permissions**: Authenticated client session
- **Special Headers**: `Authorization`, `X-Ci`, `X-Pf`, `X-Sv`, `X-Du`
- **Rate Limits**: Standard rate limit (30 requests / 60 seconds).

#### Request Details
- **Headers**: Standard `AppController` headers.
- **Query Parameters**:
  - `TYPE`: `string` (Required; currently allowed: `simple` or `FIND`)
  - `CONDITION_LIST`: `array` (Optional; applied when `TYPE=FIND`, supports filtering by status, order date, tags, target yield)
- **Sample URL**: `/fund/list?TYPE=FIND`

#### Response Details
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
- **Business Error Codes**: `system_error` (thrown if `TYPE` is omitted or neither `simple` nor `FIND`).

#### Processing Flow & Transactions
- **Flow**:
  1. Validates `TYPE` parameter.
  2. If `TYPE == 'simple'`, calls `FundModel::fundListSmall()` returning simplified code and name pairs.
  3. If `TYPE == 'FIND'`, calls `FundService::fundListForFind($condition_list)`, compiling fund tags, application totals, and sale status.
- **Transaction Boundaries**: Read-only.
- **Idempotency**: Idempotent.

#### Side Effects & Dependencies
- **Logging**: Access logs.
- **Dependencies**: `PST_FUND`, `PST_FUND_TAG`, `PST_FUND_APPRICATION`.

---

### 3. Fund Detail

#### Identification & Management
- **API Name / ID**: Fund Detail (`ST-API-FND-002`)
- **Controller - Service**: `App\Controllers\FundController::Detail` -> `App\Service\FundService::detail` & `PrientryService::getEntryElected`
- **Source File / Class**: [`App/Controllers/FundController.php::Detail`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/FundController.php#L60-L72)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Provides comprehensive details for a specific fund, including investment overview, asset schedule, yield, contract terms, lottery entry status, and whether the logged-in user won priority allocation (*Elected status*).

#### Endpoint & Environment
- **Method**: `GET`
- **URL**: `/fund/detail` (Internal rewrite: `/index.php?c=fund&a=detail`)
- **Environment**: All environments
- **Version**: v1

#### Authentication & Authorization
- **Auth Type**: Bearer Token
- **Permissions**: Authenticated client session
- **Special Headers**: `Authorization`, `X-Ci`, `X-Pf`, `X-Sv`, `X-Du`
- **Rate Limits**: Standard rate limit.

#### Request Details
- **Query Parameters**:
  - `FUND_CD`: `string` (Required; fund identifier code)
- **Sample URL**: `/fund/detail?FUND_CD=FND001`

#### Response Details
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
- **Business Error Codes**: `fund_not_exist`

#### Processing Flow & Dependencies
- **Flow**:
  1. Validates fund existence and `OPEN_FLG = 1`.
  2. Resolves current fund lifecycle period (Primary subscription, Lottery entry, or Secondary trading) via `FundHelper::fixFund`.
  3. Checks `PrientryService` for lottery win status (`PST_PRI_ENTRY_ELECTED`).
- **Dependencies**: `PST_FUND`, `PST_PRI_ENTRY_ELECTED`, `PST_PRICE`.

---

### 4. Order History List

#### Identification & Management
- **API Name / ID**: Order History List (`ST-API-HIS-001`)
- **Controller - Service**: `App\Controllers\HistoryController::OrderList`
- **Source File / Class**: [`App/Controllers/HistoryController.php::OrderList`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/HistoryController.php#L18-L77)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Lists all un-executed or cancellable buy/sell orders for the current user, calculating whether each order can still be cancelled based on current date and trading cut-off deadlines.

#### Endpoint & Environment
- **Method**: `POST` (receives filtering payload via JSON body)
- **URL**: `/history/order/list` (Internal rewrite: `/index.php?c=history&a=orderList`)
- **Environment**: All environments
- **Version**: v1

#### Authentication & Authorization
- **Auth Type**: Bearer Token
- **Permissions**: Authenticated client session
- **Special Headers**: `Authorization`, `X-Ci`, `X-Pf`, `X-Sv`, `X-Du`
- **Rate Limits**: Standard rate limit.

#### Request Details
- **Body Schema**:
  ```json
  {
    "ORDER_D_TO": "string (YYYY-MM-DD, optional, defaults to CURRENT_D)",
    "FUND_CD": "string (optional)",
    "ORDER_OBJECT": "array of string (optional: BUY, TRADE-BUY, TRADE-SELL)"
  }
  ```
- **Sample Payload**:
  ```json
  {
    "ORDER_D_TO": "2026-09-09"
  }
  ```

#### Response Details
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

#### Processing Flow & Dependencies
- **Flow**: Retrieves unexecuted orders from `PST_BUY_ORDER`, joins with `PST_FUND`, checks cut-off deadline (`pfund.order.deadline`) in `ESYS_CONFIG` via `BuyService::orderCheckCancelValid` / `TradeService::orderCheckCancelValid`.
- **Dependencies**: `PST_BUY_ORDER`, `PST_BASE_DATE`, `ESYS_CONFIG`.

---

### 5. Trade Execution History List

#### Identification & Management
- **API Name / ID**: Trade Execution History List (`ST-API-HIS-002`)
- **Controller - Service**: `App\Controllers\HistoryController::TradeList`
- **Source File / Class**: [`App/Controllers/HistoryController.php::TradeList`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/HistoryController.php#L79-L157)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Retrieves historical executed trades (約定履歴 - *Yakujo Rireki*), including primary subscription purchases, secondary buy/sell transactions, depot transfers (入庫/出庫), and redemptions.

#### Endpoint & Environment
- **Method**: `POST`
- **URL**: `/history/trade/list` (Internal rewrite: `/index.php?c=history&a=tradeList`)
- **Environment**: All environments
- **Version**: v1

#### Request & Response Details
- **Body Schema**:
  ```json
  {
    "D_TYPE": "string (TRADE_D | SETTLEMENT_D, defaults to TRADE_D)",
    "TRADE_OBJECT": "array of string (BUY, TRADE-BUY, TRADE-SELL, IN-DEPOT, OUT-DEPOT, REFUND)",
    "D_FROM": "string (YYYY-MM-DD, defaults to 1 month ago)",
    "D_TO": "string (YYYY-MM-DD, defaults to CURRENT_D)"
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

### 6. Dividend & Refund History List

#### Identification & Management
- **API Name / ID**: Dividend & Refund History List (`ST-API-HIS-003`)
- **Controller - Service**: `App\Controllers\HistoryController::DividendRefundList`
- **Source File / Class**: [`App/Controllers/HistoryController.php::DividendRefundList`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/HistoryController.php#L159-L210)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Fetches distribution payout records (分配金 - *Bunpaikin*) and redemption payout records (償還金 - *Shokankin*) for the logged-in client.

#### Endpoint & Environment
- **Method**: `POST`
- **URL**: `/history/dividend_refund/list` (Internal rewrite: `/index.php?c=history&a=dividendRefundList`)
- **Environment**: All environments
- **Version**: v1

#### Request & Response Details
- **Body Schema**:
  ```json
  {
    "TRADE_OBJECT": "array of string (DIVIDEND, REFUND)",
    "D_FROM": "string (YYYY-MM-DD)",
    "D_TO": "string (YYYY-MM-DD)"
  }
  ```
- **Response Schema**: Returns array `TRADE_LIST` containing `SUMMARY_TYPE`, `PAYMENT_D`, `QTY`, `TRADE_AMOUNT`, `TRADE_PER_PRICE`, and tax withheld.
- **Dependencies**: `PST_ALL_TRADE_HIST`, `PST_BASE_DATE`.

---

### 7. Settlement History List

#### Identification & Management
- **API Name / ID**: Settlement History List (`ST-API-HIS-004`)
- **Controller - Service**: `App\Controllers\HistoryController::AllList`
- **Source File / Class**: [`App/Controllers/HistoryController.php::AllList`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/HistoryController.php#L215-L284)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Returns a unified chronological settlement ledger (精算履歴 - *Seisan Rireki*), combining security trades, dividends, redemptions, bank deposits, and withdrawals where the settlement date is on or before the current base date.

#### Endpoint & Environment
- **Method**: `POST`
- **URL**: `/history/all/list` (Internal rewrite: `/index.php?c=history&a=allList`)
- **Environment**: All environments
- **Version**: v1

#### Request & Response Details
- **Body Schema**:
  ```json
  {
    "TRADE_OBJECT": "array of string (BUY, TRADE-BUY, TRADE-SELL, DIVIDEND, REFUND, IN, OUT)",
    "D_FROM": "string (YYYY-MM-DD)",
    "D_TO": "string (YYYY-MM-DD)"
  }
  ```
- **Response Details**: Returns `TRADE_LIST` items decorated with UI indicators such as `COLOR` (e.g., `#FFBBC3` for purchases, `#A0CAFA` for sales/redemptions, `#FFEE99` for cash deposits/withdrawals) and `SUMMARY_TYPE_TEXT`.
- **Dependencies**: `ECO_TRADE_HIST`, `ECO_BASE_DATE`.

---

### 8. Income Tax Support Data

#### Identification & Management
- **API Name / ID**: Income Tax Support Data (`ST-API-HIS-005`)
- **Controller - Service**: `App\Controllers\HistoryController::Incometax`
- **Source File / Class**: [`App/Controllers/HistoryController.php::Incometax`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/HistoryController.php#L289-L547)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Provides comprehensive tax calculation support data for filing Japanese annual income tax returns (確定申告サポート資料 - *Kakutei Shinkoku Support Shiryo*). Aggregates annual dividend income, capital gains/losses from ST sales (譲渡損益), redemption amounts, miscellaneous income (雑所得), withholding taxes, and supplier/issuer addresses.

#### Endpoint & Environment
- **Method**: `GET` / `POST`
- **URL**: `/history/incometax` (Internal rewrite: `/index.php?c=history&a=incometax`)
- **Environment**: All environments
- **Version**: v1

#### Request Details
- **Parameters**: `YEAR`: `string` (Optional, 4-digit year e.g. `2025`; defaults to most recent processed year)

#### Response Details
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

### 9. Contract Document List

#### Identification & Management
- **API Name / ID**: Contract Document List (`ST-API-ORD-001`)
- **Controller - Service**: `App\Controllers\OrderController::ContractList` -> `App\Service\ContractService::contractList`
- **Source File / Class**: [`App/Controllers/OrderController.php::ContractList`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/OrderController.php#L15-L26)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Retrieves electronic statutory disclosure documents (目論見書 - Prospectus, 契約締結前交付書面 - Pre-contract document, 契約書 - Investment contract) for a specific fund, detailing version, document type, and whether the client has agreed to them.

#### Endpoint & Environment
- **Method**: `GET` / `POST`
- **URL**: `/order/contract/list` (Internal rewrite: `/index.php?c=order&a=contractList`)
- **Environment**: All environments
- **Version**: v1

#### Request Details
- **Parameters**: `FUND_CD`: `string` (Required)

#### Response Details
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
- **Business Error Codes**: `fund_not_exist`
- **Dependencies**: `PST_CONTRACT_PDF`, `PST_CONTRACT_LOG`, `PST_FUND`.

---

### 10. Contract Document Read

#### Identification & Management
- **API Name / ID**: Contract Document Read (`ST-API-ORD-002`)
- **Controller - Service**: `App\Controllers\OrderController::ContractRead` -> `App\Service\ContractService::contractRead`
- **Source File / Class**: [`App/Controllers/OrderController.php::ContractRead`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/OrderController.php#L28-L37)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Generates a temporary pre-signed AWS S3 URL for downloading/viewing a contract PDF and records that the user has opened/read the document in Redis.

#### Endpoint & Environment
- **Method**: `GET` / `POST`
- **URL**: `/order/contract/read` (Internal rewrite: `/index.php?c=order&a=contractRead`)
- **Environment**: All environments
- **Version**: v1

#### Request Details
- **Parameters**:
  - `FUND_CD`: `string` (Required)
  - `CONTRACT_TYPE`: `string` (Required; `1`, `2`, or `3`)
  - `CONTRACT_VERSION`: `string` (Required)

#### Response Details
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
- **Business Error Codes**: `pdf_not_exist`

#### Side Effects
- **Caching**: Sets Redis hash `CR:<CLIENT_ID>:<CONTRACT_NUM>` with key `<CONTRACT_VERSION>` containing the document metadata and sets TTL to 24 hours (86,400s).

---

### 11. Contract Document Accept

#### Identification & Management
- **API Name / ID**: Contract Document Accept (`ST-API-ORD-003`)
- **Controller - Service**: `App\Controllers\OrderController::ContractAccept` -> `App\Service\ContractService::contractAccept`
- **Source File / Class**: [`App/Controllers/OrderController.php::ContractAccept`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/OrderController.php#L39-L48)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Confirms the investor's statutory acceptance/consent of the disclosure document. Verifies that the document was actually opened and read via Redis verification before persisting agreement records.

#### Endpoint & Environment
- **Method**: `POST`
- **URL**: `/order/contract/accept` (Internal rewrite: `/index.php?c=order&a=contractAccept`)
- **Environment**: All environments
- **Version**: v1

#### Request Details
- **Body Schema**:
  ```json
  {
    "FUND_CD": "string (Required)",
    "CONTRACT_TYPE": "string (Required: 1, 2, or 3)",
    "CONTRACT_VERSION": "string (Required)"
  }
  ```

#### Response Details
- **Schema**:
  ```json
  {
    "STATUS": "OK",
    "ERROR": null,
    "DATA": []
  }
  ```
- **Business Error Codes**: `fund_not_exist`, `pdf_not_exist`, `pdf_not_read`, `system_error`

#### Processing Flow & Transactions
- **Flow**:
  1. Checks Redis key `CR:<CLIENT_ID>:<CONTRACT_NUM>` for `<CONTRACT_VERSION>`. Throws `pdf_not_read` if missing.
  2. For contract type `3` (Partnership Agreement), ensures the user has not previously signed another version.
  3. Begins database transaction (`AppModel::beginTransaction()`).
  4. Inserts agreement into `PST_CONTRACT_LOG`.
  5. Records legal delivery into `ECO_ELECTRONIC_DELIVERY`.
  6. Commits database transaction.
- **Transaction Boundaries**: Atomic across `PST_CONTRACT_LOG` and `ECO_ELECTRONIC_DELIVERY`.

---

### 12. Text Document List

#### Identification & Management
- **API Name / ID**: Text Document List (`ST-API-ORD-004`)
- **Controller - Service**: `App\Controllers\OrderController::TextdocumentList` -> `App\Service\TextDocumentService::textdocumentList`
- **Source File / Class**: [`App/Controllers/OrderController.php::TextdocumentList`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/OrderController.php#L50-L61)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Retrieves required electronic text terms/covenants that must be acknowledged before submitting an order.

#### Endpoint & Environment
- **Method**: `GET`
- **URL**: `/order/textdocument/list` (Internal rewrite: `/index.php?c=order&a=textdocumentList`)
- **Environment**: All environments
- **Version**: v1

#### Request & Response Details
- **Parameters**: `FUND_CD`: `string` (Required)
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
          "CONTENT": "string (formatted with HTML br tags)",
          "AGREE_FLG": "string"
        }
      ]
    }
  }
  ```
- **Dependencies**: `PST_TEXT_DOCUMENT`, `PST_TEXT_DOCUMENT_LOG`.

---

### 13. Text Document Accept

#### Identification & Management
- **API Name / ID**: Text Document Accept (`ST-API-ORD-005`)
- **Controller - Service**: `App\Controllers\OrderController::TextdocumentAccept` -> `App\Service\TextDocumentService::textDocumentAccept`
- **Source File / Class**: [`App/Controllers/OrderController.php::TextdocumentAccept`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/OrderController.php#L63-L72)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Records the investor's consent to text disclosure terms.

#### Endpoint & Environment
- **Method**: `POST`
- **URL**: `/order/textdocument/accept` (Internal rewrite: `/index.php?c=order&a=textdocumentAccept`)
- **Environment**: All environments
- **Version**: v1

#### Request Details
- **Body Schema**:
  ```json
  {
    "FUND_CD": "string (Required)",
    "TEXTDOCUMENT_LIST": [
      {
        "TEXT_DOCUMENT_SEQ_NO": "string (Required)"
      }
    ]
  }
  ```

#### Response Details
- **Schema**: `{"STATUS": "OK", "ERROR": null, "DATA": []}`
- **Processing Flow**: Starts DB transaction, writes records to `PST_TEXT_DOCUMENT_LOG`, commits.

---

### 14. Order Input Information

#### Identification & Management
- **API Name / ID**: Order Input Information (`ST-API-ORD-006`)
- **Controller - Service**: `App\Controllers\OrderController::BuyInput` -> `BuyService::buyInputBuy` / `TradeService::tradeBuyInput` / `TradeService::tradeSellInput`
- **Source File / Class**: [`App/Controllers/OrderController.php::BuyInput`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/OrderController.php#L74-L104)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Prepares order entry screens for primary subscription buying (`BUY`), lottery winner buying (`ENTRY-BUY`), secondary market buying (`TRADE-BUY`), or secondary market selling (`TRADE-SELL`). Computes unit price, minimum/maximum order quantities, commission tiers, consumption tax, and available cash or sellable ST balance.

#### Endpoint & Environment
- **Method**: `GET` / `POST`
- **URL**: `/order/buy/input` (Internal rewrite: `/index.php?c=order&a=buyInput`)
- **Environment**: All environments
- **Version**: v1

#### Request Details
- **Parameters / Body**:
  - `FUND_CD`: `string` (Required)
  - `TRANSACTION_TYPE`: `string` (Optional, defaults to `BUY`; options: `BUY`, `ENTRY-BUY`, `TRADE-BUY`, `TRADE-SELL`)

#### Response Details
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
- **Business Error Codes**: `fund_not_exist`, `buy_order_un_right_time`, `entry_order_period_error`, `entry_not_elected`, `entry_buy_already_purchased`.

---

### 15. Order Confirmation

#### Identification & Management
- **API Name / ID**: Order Confirmation (`ST-API-ORD-007`)
- **Controller - Service**: `App\Controllers\OrderController::BuyConfirm` -> `BuyService::buyConfirm` / `TradeService::tradeBuyConfirm` / `TradeService::tradeSellConfirm`
- **Source File / Class**: [`App/Controllers/OrderController.php::BuyConfirm`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/OrderController.php#L106-L143)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Validates the proposed order (order quantity, multiple of unit quantity, remaining fund quota, daily investor limit, available funds/units, and non-disclosure agreements). Caches the verified order preview in Redis (`TMP_ORDER:<order_id>`) for subsequent execution.

#### Endpoint & Environment
- **Method**: `POST`
- **URL**: `/order/buy/confirm` (Internal rewrite: `/index.php?c=order&a=buyConfirm`)
- **Environment**: All environments
- **Version**: v1

#### Authentication & Authorization
- **Auth Type**: Bearer Token + `X-Ct` Anti-Tamper Token (Required in non-local environments; validated against Redis `ct:<token>`).

#### Request Details
- **Headers**: Standard `AppController` headers + `X-Ct: <token>`.
- **Body Schema**:
  ```json
  {
    "TRANSACTION_TYPE": "string (Required: BUY, ENTRY-BUY, TRADE-BUY, or TRADE-SELL)",
    "FUND_CD": "string (Required)",
    "ORDER_QTY": "string (Required, integer string)",
    "CONFIRM_INSIDER_CHECK": "integer (optional, required if user is flagged insider)"
  }
  ```

#### Response Details
- **Response Schema**: Returns full calculated figures including `ORDER_AMOUNT`, `COMMISSION_AMOUNT`, `TAX_AMOUNT`, `TOTAL_PAYMENT_AMOUNT`, and temporary order key.
- **Business Error Codes**: `buy_order_input`, `buy_order_min_qty`, `buy_order_unit_qty`, `buy_order_max_qty`, `buy_order_buyable_cash`, `buy_fund_max_qty_shortage`, `buy_order_un_confirmed_doc`, `fund_user_in_stock_error`.

#### Side Effects
- **Caching**: Stores order preview in Redis `TMP_ORDER:<order_id>` with short TTL.
- **Token Deletion**: Consumes and deletes `X-Ct` token (`ct:<token>`) from Redis upon completion.

---

### 16. Order Execution Complete

#### Identification & Management
- **API Name / ID**: Order Execution Complete (`ST-API-ORD-008`)
- **Controller - Service**: `App\Controllers\OrderController::BuyComplete` -> `BuyService::buyComplete` / `TradeService::tradeBuyComplete` / `TradeService::tradeSellComplete`
- **Source File / Class**: [`App/Controllers/OrderController.php::BuyComplete`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/OrderController.php#L145-L185)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Finalizes and commits the financial order into the system. Validates trade password or PIN, updates investor cash balance or security balance via the ledger engine (`netsec/common-balance`), creates the order record (`PST_BUY_ORDER` or `PST_PROP_TRADE_ORDER`), updates allocation cumulative counts, and sends trade notifications.

#### Endpoint & Environment
- **Method**: `POST`
- **URL**: `/order/buy/complete` (Internal rewrite: `/index.php?c=order&a=buyComplete`)
- **Environment**: All environments
- **Version**: v1

#### Authentication & Authorization
- **Auth Type**: Bearer Token + `X-Ct` Anti-Tamper Token.

#### Request Details
- **Body Schema**:
  ```json
  {
    "TRANSACTION_TYPE": "string (Required: BUY, ENTRY-BUY, TRADE-BUY, or TRADE-SELL)",
    "FUND_CD": "string (Required)",
    "ORDER_QTY": "string (Required)",
    "PIN": "string (Required: 4-8 digit transaction PIN)"
  }
  ```

#### Response Details
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
- **Business Error Codes**: `pin_error`, `pass_locked`, `E005-0013`, `E005-0016`, `buy_order_buyable_cash`, `fund_in_stock_error`, `self_cash_balance_update_failed`.

#### Processing Flow & Transactions
- **Flow**:
  1. Validates `X-Ct` token.
  2. Verifies transaction PIN / trading password against `ECO_USER` (tracks consecutive failures; locks account upon exceeding limit with `E005-0013`).
  3. Starts atomic database transaction (`AppModel::beginTransaction()`).
  4. Interacts with `CashAbleModel` / `common-balance` ledger library to reserve/transfer cash balances.
  5. Inserts order into `PST_BUY_ORDER` with status `0` (awaiting settlement).
  6. Updates `PST_FUND_APPRICATION` counters.
  7. Inserts transaction notification queue into `PST_TRADE_NOTIFICATION`.
  8. Commits database transaction (`AppModel::commitTransaction()`).
  9. Invalidates Redis preview cache and deletes `X-Ct`.
- **Transaction Boundaries**: Full ACID transaction wrapping balance reservation, order insertion, and notification queueing.

---

### 17. Order Cancellation

#### Identification & Management
- **API Name / ID**: Order Cancellation (`ST-API-ORD-009`)
- **Controller - Service**: `App\Controllers\OrderController::Cancel` -> `BuyService::buyCancel` / `TradeService::buyCancel` / `TradeService::sellCancel`
- **Source File / Class**: [`App/Controllers/OrderController.php::Cancel`](/st-api/App/Controllers/OrderController.php#L187-L224)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Cancels an active, un-executed order (primary subscription order, secondary buy order, or secondary sell order). Releases reserved cash back into the investor's purchasing balance or restores held ST units.

#### Endpoint & Environment
- **Method**: `POST`
- **URL**: `/order/cancel` (Internal rewrite: `/index.php?c=order&a=cancel`)
- **Environment**: All environments
- **Version**: v1

#### Authentication & Authorization
- **Auth Type**: Bearer Token + `X-Ct` Anti-Tamper Token.

#### Request Details
- **Body Schema**:
  ```json
  {
    "CANCEL_TYPE": "string (Required: BUY, TRADE-BUY, or TRADE-SELL)",
    "ORDER_NUM": "string (Required: order sequence identifier)"
  }
  ```

#### Response Details
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
- **Business Error Codes**: `order_not_exist`, `order_cancel_invalid`, `buy_order_un_right_time`.

#### Processing Flow & Transactions
- **Flow**:
  1. Validates order status is unexecuted (`ORDER_STATUS = 0`).
  2. Verifies that cancellation is requested prior to daily cut-off deadline (`pfund.order.deadline`).
  3. Starts atomic DB transaction.
  4. Updates `ORDER_STATUS = 10` (Cancelled).
  5. Unfreezes reserved cash via `CashAbleModel` or restores ST position in `FundRealBalanceModel`.
  6. Commits DB transaction.
  7. Deletes `X-Ct`.

---

### 18. Fund Insider Confirmation

#### Identification & Management
- **API Name / ID**: Fund Insider Confirmation (`ST-API-ORD-010`)
- **Controller - Service**: `App\Controllers\OrderController::insiderconfirm` -> `App\Service\InsiderConfirmService::insiderConfirm`
- **Source File / Class**: [`App/Controllers/OrderController.php::insiderconfirm`](/st-api/App/Controllers/OrderController.php#L229-L237)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Compliance check that verifies whether the logged-in user is registered as a corporate insider (役員・主要株主 - Officer / Major Shareholder) for the issuing entity or affiliated companies of the targeted fund.

#### Endpoint & Environment
- **Method**: `GET` / `POST`
- **URL**: `/order/insiderconfirm` (Internal rewrite: `/index.php?c=order&a=insiderconfirm`)
- **Environment**: All environments
- **Version**: v1

#### Request & Response Details
- **Parameters**: `FUND_CD`: `string` (Required)
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

### 19. Priority Entry MyPage

#### Identification & Management
- **API Name / ID**: Priority Entry MyPage (`ST-API-PRI-001`)
- **Controller - Service**: `App\Controllers\PrientryController::mypage` -> `App\Service\PrientryService::fundListForPreSale`
- **Source File / Class**: [`App/Controllers/PrientryController.php::mypage`](/st-api/App/Controllers/PrientryController.php#L17-L24)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Retrieves funds currently in the pre-subscription lottery entry phase (優先申込エントリー - *Yusen Moshikomi Entry*) and funds currently undergoing lottery drawing (抽選期間中).

#### Endpoint & Environment
- **Method**: `GET`
- **URL**: `/prientry/mypage` (Internal rewrite: `/index.php?c=prientry&a=mypage`)
- **Environment**: All environments
- **Version**: v1

#### Response Details
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

### 20. Priority Entry Fund Detail

#### Identification & Management
- **API Name / ID**: Priority Entry Fund Detail (`ST-API-PRI-002`)
- **Controller - Service**: `App\Controllers\PrientryController::fund` -> `FundService::detail` & `PrientryService::getEntry`
- **Source File / Class**: [`App/Controllers/PrientryController.php::fund`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/PrientryController.php#L31-L44)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Provides detailed specifications for a fund in the priority lottery entry phase, along with the user's existing entry status if already submitted.

#### Endpoint & Environment
- **Method**: `GET`
- **URL**: `/prientry/fund/{FUND_CD}` (Internal rewrite: `/index.php?c=prientry&a=fund&FUND_CD=$1`)
- **Environment**: All environments
- **Version**: v1

#### Request Details
- **Route / Query Parameter**: `FUND_CD`: `string` (Required)

#### Response Details
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

### 21. Priority Entry Application

#### Identification & Management
- **API Name / ID**: Priority Entry Application (`ST-API-PRI-003`)
- **Controller - Service**: `App\Controllers\PrientryController::entry` -> `App\Service\PrientryService::entryComplete`
- **Source File / Class**: [`App/Controllers/PrientryController.php::entry`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/PrientryController.php#L51-L67)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Submits a lottery application for priority allocation of a fund before general public subscription opens. Validates entry period, minimum/maximum entry units, and residency card validity.

#### Endpoint & Environment
- **Method**: `POST`
- **URL**: `/prientry/entry/{FUND_CD}` (Internal rewrite: `/index.php?c=prientry&a=entry&FUND_CD=$1`)
- **Environment**: All environments
- **Version**: v1

#### Authentication & Authorization
- **Auth Type**: Bearer Token + `X-Ct` Anti-Tamper Token.

#### Request Details
- **Route Parameter**: `FUND_CD`: `string` (Required)
- **Body Schema**:
  ```json
  {
    "ORDER_QTY": "string (Required: number of units requested)"
  }
  ```

#### Response Details
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
- **Business Error Codes**: `entry_input_qty`, `entry_min_qty`, `entry_bought`, `entry_fund_not_start`, `entry_fund_end`, `entry_max_qty_fund`, `entry_user_status_stop_buy_flg`.

#### Processing Flow & Transactions
- **Flow**: Starts DB transaction, creates entry record in `PST_PRI_ENTRY`, updates `PST_PRI_ENTRY_SUM`, commits.
- **Side Effects**: Consumes `X-Ct`.

---

### 22. Priority Entry Update Quantity

#### Identification & Management
- **API Name / ID**: Priority Entry Update Quantity (`ST-API-PRI-004`)
- **Controller - Service**: `App\Controllers\PrientryController::update` -> `App\Service\PrientryService::updateComplete`
- **Source File / Class**: [`App/Controllers/PrientryController.php::update`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/PrientryController.php#L74-L90)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Modifies the applied unit count of an existing priority lottery entry while the entry window remains open.

#### Endpoint & Environment
- **Method**: `POST`
- **URL**: `/prientry/update/{FUND_CD}` (Internal rewrite: `/index.php?c=prientry&a=update&FUND_CD=$1`)
- **Environment**: All environments
- **Version**: v1

#### Authentication & Authorization
- **Auth Type**: Bearer Token + `X-Ct` Anti-Tamper Token.

#### Request Details
- **Route Parameter**: `FUND_CD`: `string` (Required)
- **Body Schema**:
  ```json
  {
    "ORDER_QTY": "string (Required: new unit count)"
  }
  ```

#### Response Details
- **Schema**: Returns updated fund metadata (`FUND_CD`, `FUND_NM`, `ICON`).
- **Business Error Codes**: `entry_not_applied`, `entry_not_update`, `entry_fund_end`.

---

### 23. Priority Entry Cancellation

#### Identification & Management
- **API Name / ID**: Priority Entry Cancellation (`ST-API-PRI-005`)
- **Controller - Service**: `App\Controllers\PrientryController::cancel` -> `App\Service\PrientryService::cancelComplete`
- **Source File / Class**: [`App/Controllers/PrientryController.php::cancel`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/PrientryController.php#L97-L113)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Cancels an existing priority lottery entry before the entry window closes.

#### Endpoint & Environment
- **Method**: `POST`
- **URL**: `/prientry/cancel/{FUND_CD}` (Internal rewrite: `/index.php?c=prientry&a=cancel&FUND_CD=$1`)
- **Environment**: All environments
- **Version**: v1

#### Authentication & Authorization
- **Auth Type**: Bearer Token + `X-Ct` Anti-Tamper Token.

#### Request Details
- **Route Parameter**: `FUND_CD`: `string` (Required)

#### Response Details
- **Schema**: Returns cancelled fund metadata (`FUND_CD`, `FUND_NM`, `ICON`).
- **Business Error Codes**: `entry_not_applied`, `entry_fund_end`.

---

### 24. SendGrid Webhook Notification

#### Identification & Management
- **API Name / ID**: SendGrid Webhook Notification (`ST-API-NOT-001`)
- **Controller - Service**: `App\Controllers\NotificationController::sendgridWebhook` -> `App\Service\WebhookService::sendGrid`
- **Source File / Class**: [`App/Controllers/NotificationController.php::sendgridWebhook`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/NotificationController.php#L13-L37)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Receives asynchronous delivery status webhooks from SendGrid for transaction-related emails (order confirmation, trade execution slips, contract delivery notices), verifying ECDSA cryptographic signatures and updating mail statuses in `PST_TRADE_NOTIFICATION`.

#### Endpoint & Environment
- **Method**: `POST`
- **URL**: `/webhook/transfer-notification/sendgrid/{x-ci}` (Internal rewrite: `/index.php?c=notification&a=sendgridWebhook&x-ci=$1`)
- **Environment**: All environments
- **Version**: v1

#### Authentication & Authorization
- **Auth Type**: ECDSA Webhook Signature Verification (`SendGrid\EventWebhook\EventWebhook`).
- **Permissions**: None required (Public webhook). Extends `BraveController` directly, bypassing `AppController` user session checks.
- **Special Headers**:
  - `X-Twilio-Email-Event-Webhook-Signature`: ECDSA signature string (Required)
  - `X-Twilio-Email-Event-Webhook-Timestamp`: Event timestamp (Required)

#### Request Details
- **Route Parameter**: `x-ci`: Company ID (resolves tenant database context)
- **Body Schema**: JSON array of SendGrid Event objects:
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

#### Response Details
- **Status Codes**: `200 OK` on success, `401 Unauthorized` on signature failure.
- **Business Error Codes**: `sendgrid_verification_error`, `unknown_error`.

#### Processing Flow & Transactions
- **Flow**:
  1. Reads `transfer_notification.sendgrid_public_verification_key` from config.
  2. Verifies ECDSA signature against raw HTTP payload.
  3. Starts atomic DB transaction.
  4. Updates `PST_TRADE_NOTIFICATION` mail status (`MAIL_STATUS`: 1 = processed, 2 = dropped/bounce, 10 = delivered, 11 = open/click) and records processed timestamps.
  5. Commits DB transaction.

---

### 25. S3 Icon Image Proxy

#### Identification & Management
- **API Name / ID**: S3 Icon Image Proxy (`ST-API-MED-001`)
- **Controller - Service**: `App\Controllers\UnauthCommonController::icon`
- **Source File / Class**: [`App/Controllers/UnauthCommonController.php::icon`](file:///home/haidang/Works/ownersbook+/st-api/App/Controllers/UnauthCommonController.php#L13-L57)
- **Status**: Implemented / Active
- **Date / Version**: Current Master / v1.0

#### Business Description
Streams public fund branding assets, badges, and icons stored in AWS S3 as Base64-encoded Data URIs to web and mobile clients.

#### Endpoint & Environment
- **Method**: `GET`
- **URL**: `/s3/{x-ci}/{icon-key}` (Internal rewrite: `/index.php?c=unauthCommon&a=icon&x-ci=$1&icon-key=$2`)
- **Environment**: All environments
- **Version**: v1

#### Authentication & Authorization
- **Auth Type**: Bearer Token (inherits `AppController` session verification).
- **Security Check**: `icon-key` must start with `public` and possess a valid image file extension (`png`, `jpeg`, `jpg`, `gif`, `svg`, etc.).

#### Request Details
- **Parameters**:
  - `x-ci`: Company ID (Required)
  - `icon-key`: S3 object key path (Required, e.g. `public/funds/fnd01_icon.png`)

#### Response Details
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
- **Business Error Codes**: `unknown_error`

---

## 4. Notes & Important Architectural Caveats

1. **Nginx Route Discrepancies (Crucial for Newcomers)**:
   - In `local-docker/etc/local/nginx/pfund-api.conf`, rewrite rules exist for `/order/sell/input`, `/order/sell/confirm`, and `/order/sell/complate`. However, there are **NO** corresponding `sellInput`, `sellConfirm`, or `sellComplate` methods in `OrderController.php`.
   - **Correct Usage**: All secondary market sell operations must be called via `/order/buy/input`, `/order/buy/confirm`, and `/order/buy/complete` with `TRANSACTION_TYPE: 'TRADE-SELL'`. Calling the `/order/sell/*` paths directly triggers a `404 router error`.
   - Similarly, `/history/trade/detail`, `/history/order/detail`, `/history/dividend/detail`, and `/history/refund/detail` in the Nginx config are unhandled rewrite stubs. All historical queries must use the respective list APIs (`/history/trade/list`, `/history/order/list`, etc.).

2. **Mandatory Decimal Math Rule (`ext-bcmath`)**:
   - As an institutional financial trading service, native PHP floating-point arithmetic (`float`, `double`) is strictly prohibited in any calculation involving order units, prices, commissions, dividends, or taxes.
   - All arithmetic must use BCMath string functions (`bcadd`, `bcsub`, `bcmul`, `bcdiv`, `bccomp`) with explicitly declared scales.

3. **Multi-Tenancy & Database Routing (`X-Ci`)**:
   - Every request requires `X-Ci` header. The system queries `basis.BCO_COMPANY` where `COMPANY_ID = :X-Ci` to determine the database prefix. If `BCO_COMPANY` does not contain the tenant record, the API fails with `Unknown database '_pfund'`.

4. **Two-Step Order Submission & Anti-Tamper Tokens (`X-Ct`)**:
   - Sensitive write operations (`/order/buy/confirm`, `/order/buy/complete`, `/order/cancel`, `/prientry/entry/*`, `/prientry/update/*`, `/prientry/cancel/*`) require the `X-Ct` header in non-local environments.
   - The front-end fetches or generates the token in Redis (`ct:<token>`), which is verified and immediately consumed (`deleteXct()`) to prevent replay attacks and double submissions.

5. **Session Architecture & No-DB Token Validation**:
   - `st-api` does not query the SQL database to authenticate Bearer tokens on each HTTP request. It queries Redis directly (`TK:<COMPANY_SEQ_NO>:<token>`), achieving $O(1)$ sub-millisecond authentication without overloading the relational database.
