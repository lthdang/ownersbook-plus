# Implementation Plan: st-api API Specification Documentation

## 1. Overview & Goal
The objective is to analyze all APIs implemented within `st-api` (Fund Core API / Security Token Trading API) and generate a comprehensive reference document (`api-informations.md`) to onboard new developers quickly and accurately.

The synthesized documentation must strictly reflect the actual implementation in `st-api`, following the requested output structure, with exact parameter names, schemas, authentication mechanisms, ledger dependencies, and business rules.

---

## 2. Project Architecture & Technical Context

- **Project Name**: `st-api` (ST-FRONT-API / Fund Core API)
- **Primary Language**: PHP 8.3 (tested with PHP 8.3.16 in `st-app-pfund` container)
- **Framework & Architecture**: Custom Brave Framework (MVC-like framework with `BraveDispatcher`, `BraveController`, `AppController`), Service layer (`App\Service`), Model layer (`App\Models`)
- **Database & Data Access**: 
  - Abstraction: `adodb/adodb-php: 5.22.7` with `BraveMySQLBuilder`
  - Multi-tenancy routing: Dynamic tenant database resolution via header `X-Ci` (`basis.BCO_COMPANY` maps `COMPANY_ID` to `{prefix}_pfund` and company databases)
  - Databases involved: `basis` (system configurations & companies), `{prefix}_pfund` (fund & ST trading data), `{prefix}` (user accounts & ledger data)
- **Ledger Engine**: `netsec/common-balance` (`CashAbleModel`, `CashRealBalanceModel`) for deposit and trading balance management with arbitrary-precision arithmetic (`ext-bcmath`)
- **Caching & Session Storage**: Redis (OAuth2 Bearer token sessions `TK:<company_seq>:<token>`, rate limiting counters `RATE_LIMIT:<user_seq>:<window>`, transaction tokens `ct:<token>`, order previews `TMP_ORDER:<order_id>`)
- **Telemetry & Monitoring**: Sentry SDK (`sentry/sdk: ^3.1`), Monolog (`monolog/monolog: ^2.3`), Fluentd (`fluent/logger: ^1.0`)
- **Reverse Proxy / Routing**: Nginx rewrites (`local-docker/etc/local/nginx/pfund-api.conf`) translating REST endpoints into Brave query parameters (`/index.php?c=<controller>&a=<action>`).

---

## 3. Scope of Analysis (20 Implemented APIs)

Based on exhaustive analysis of `App/Controllers/`, `App/Service/`, and `App/Models/`, exactly 20 APIs are implemented across 7 controllers:

| No. | API Name / ID | Method | Endpoint / URL | Controller Method | Module | Status |
|---|---|---|---|---|---|---|
| 1 | Asset Summary | GET | `/asset/summary` | `AssetController::summary` | Asset / Portfolio | Implemented |
| 2 | Fund List | GET | `/fund/list` | `FundController::List` | Fund | Implemented |
| 3 | Fund Detail | GET | `/fund/detail` | `FundController::Detail` | Fund | Implemented |
| 4 | Order List | POST | `/history/order/list` | `HistoryController::OrderList` | History | Implemented |
| 5 | Trade Execution List | POST | `/history/trade/list` | `HistoryController::TradeList` | History | Implemented |
| 6 | Dividend & Refund List | POST | `/history/dividend_refund/list` | `HistoryController::DividendRefundList` | History | Implemented |
| 7 | Settlement History List | POST | `/history/all/list` | `HistoryController::AllList` | History | Implemented |
| 8 | Income Tax Support Info | GET/POST | `/history/incometax` | `HistoryController::Incometax` | History / Tax | Implemented |
| 9 | Contract Document List | GET/POST | `/order/contract/list` | `OrderController::ContractList` | Order / Contract | Implemented |
| 10 | Contract Document Read | GET/POST | `/order/contract/read` | `OrderController::ContractRead` | Order / Contract | Implemented |
| 11 | Contract Document Accept | POST | `/order/contract/accept` | `OrderController::ContractAccept` | Order / Contract | Implemented |
| 12 | Text Document List | GET | `/order/textdocument/list` | `OrderController::TextdocumentList` | Order / Text Document | Implemented |
| 13 | Text Document Accept | POST | `/order/textdocument/accept` | `OrderController::TextdocumentAccept` | Order / Text Document | Implemented |
| 14 | Order Input Info | GET/POST | `/order/buy/input` | `OrderController::BuyInput` | Order / Trading | Implemented |
| 15 | Order Confirmation | POST | `/order/buy/confirm` | `OrderController::BuyConfirm` | Order / Trading | Implemented |
| 16 | Order Completion | POST | `/order/buy/complete` | `OrderController::BuyComplete` | Order / Trading | Implemented |
| 17 | Order Cancellation | POST | `/order/cancel` | `OrderController::Cancel` | Order / Trading | Implemented |
| 18 | Fund Insider Confirmation | GET/POST | `/order/insiderconfirm` | `OrderController::insiderconfirm` | Order / Compliance | Implemented |
| 19 | Priority Entry MyPage | GET | `/prientry/mypage` | `PrientryController::mypage` | Priority Entry | Implemented |
| 20 | Priority Entry Fund Detail | GET | `/prientry/fund/{FUND_CD}` | `PrientryController::fund` | Priority Entry | Implemented |
| 21 | Priority Entry Apply | POST | `/prientry/entry/{FUND_CD}` | `PrientryController::entry` | Priority Entry | Implemented |
| 22 | Priority Entry Update Qty | POST | `/prientry/update/{FUND_CD}` | `PrientryController::update` | Priority Entry | Implemented |
| 23 | Priority Entry Cancel | POST | `/prientry/cancel/{FUND_CD}` | `PrientryController::cancel` | Priority Entry | Implemented |
| 24 | SendGrid Webhook | POST | `/webhook/transfer-notification/sendgrid/{x-ci}` | `NotificationController::sendgridWebhook` | Notification / Webhook | Implemented |
| 25 | S3 Icon Image Proxy | GET | `/s3/{x-ci}/{icon-key}` | `UnauthCommonController::icon` | Media / Proxy | Implemented |

*(Note: In Nginx configuration, 7 routes such as `/order/sell/input`, `/order/sell/confirm`, `/order/sell/complate`, `/history/trade/detail`, `/history/order/detail`, `/history/dividend/detail`, `/history/refund/detail` exist in rewrite rules but do not have corresponding controller actions implemented in `st-api`. This will be documented as an essential note for newcomers).*

---

## 4. Implementation Steps

1. **Step 1 - Review & Verification**:
   - Verify every controller method, associated service class, and model query.
   - Trace parameters, headers, validation checks, error responses, database transactions, and Redis keys for all 25 operations.

2. **Step 2 - Generate Documentation Plan**:
   - Write this detailed plan to `/home/haidang/Works/ownersbook+/my-folder/plans/summary-api-info.md`.
   - Present the implementation plan to the user for confirmation.

3. **Step 3 - Compile Comprehensive API Documentation**:
   - Create `/home/haidang/Works/ownersbook+/my-folder/infomations/st-api/api-informations.md`.
   - Write Section 1: Header (Project Information).
   - Write Section 2: Summary Table (Index).
   - Write Section 3: Main Content (Detailed API specifications per the Output Template for each API).
   - Write Section 4: Notes (Key architecture caveats, decimal precision rules, unimplemented nginx rewrite routes, Redis token mechanics, multi-tenancy requirements).

4. **Step 4 - Verification**:
   - Verify that all files exist and match Markdown formatting.
   - Verify that Japanese domain terms are preserved with English explanations.
