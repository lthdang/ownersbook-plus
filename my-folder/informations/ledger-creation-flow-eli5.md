# OwnersBook+ Ledger Creation Flow & Related APIs (ELI5 Beginner Guide)

## Table of Contents
1. [What is a "Ledger"? (The Real-World Analogy)](#1-what-is-a-ledger-the-real-world-analogy)
2. [The Big Picture: Two Types of Ledgers (Type A vs. Type B)](#2-the-big-picture-two-types-of-ledgers-type-a-vs-type-b)
3. [Type A — The Statutory Ledger Creation Flow (Step-by-Step)](#3-type-a--the-statutory-ledger-creation-flow-step-by-step)
   - [Step 1: Transaction Initiation & Escrow Recording (`common-front-api`)](#step-1-transaction-initiation--escrow-recording-common-front-api)
   - [Step 2: Batch Scheduling & Invocation Trigger (`st-cli` / Airflow)](#step-2-batch-scheduling--invocation-trigger-st-cli--airflow)
   - [Step 3: Pre-Execution Validation & Calendar Check (`st-cli`)](#step-3-pre-execution-validation--calendar-check-st-cli)
   - [Step 4: Data Retrieval & Statutory Processing (`st-cli`)](#step-4-data-retrieval--statutory-processing-st-cli)
   - [Step 5: PDF/Report Generation & Formatting (`st-cli`)](#step-5-pdfreport-generation--formatting-st-cli)
   - [Step 6: S3 Archival, Directory Publishing & Cleanup (`st-cli`)](#step-6-s3-archival-directory-publishing--cleanup-st-cli)
   - [Step 7: Backoffice Access & Electronic Delivery (`st-admin` / `common-front-api`)](#step-7-backoffice-access--electronic-delivery-st-admin--common-front-api)
4. [Meet the Three Statutory Ledgers: RT003, RT004, and RT005](#4-meet-the-three-statutory-ledgers-rt003-rt004-and-rt005)
5. [Type B — The Distributed Ledger (On-Chain GoQuorum)](#5-type-b--the-distributed-ledger-on-chain-goquorum)
6. [Comprehensive API Reference Table](#6-comprehensive-api-reference-table)
7. [Points to Note & Common Gotchas](#7-points-to-note--common-gotchas)

---

## 1. What is a "Ledger"? (The Real-World Analogy)

Imagine you run a school bookstore with a savings club for students. 

Every time a student deposits pocket money, buys a textbook, or withdraws their leftover lunch allowance, you write it down in a notebook.

That special record notebook is called a **ledger**.

A ledger simply keeps track of who owns what money and which goods changed hands over time.

In our system, financial regulations require us to print official, tamper-proof versions of these records to prove that our company handled every single yen correctly.

---

## 2. The Big Picture: Two Types of Ledgers (Type A vs. Type B)

When developers talk about a "ledger" in OwnersBook+, they mean two completely different things that should never be confused:

### 1. Type A — Statutory Ledger (Our Main Subject)
- **What it is**: Official PDF and CSV financial reports required by government regulators.
- **The Analogy**: An official printed monthly bank statement stamped by an accountant.
- **How it runs**: An automated background robot called a **batch** reads our database at night and generates PDF files.
- **Key Codebase**: The `st-cli` repository.

### 2. Type B — Distributed Ledger (Secondary Component)
- **What it is**: Digital ownership tokens living on a private blockchain network called **GoQuorum**.
- **The Analogy**: A shared digital database where everyone has a verifiable copy of who owns each security token.
- **How it runs**: Smart contracts handle token transfers in real time, and we inspect them using a web screen.
- **Key Codebase**: The `st-bc-explorer` repository.

> **Crucial Rule**: Type A and Type B are completely separate. Type A does not talk directly to the blockchain, and Type B does not create government PDF documents.

---

## 3. Type A — The Statutory Ledger Creation Flow (Step-by-Step)

The statutory ledger process follows **7 sequential steps** from start to finish.

### Step 1: Transaction Initiation & Escrow Recording (`common-front-api`)
- Users open our web app or mobile app to deposit money, buy investment shares, or withdraw cash.
- A **BFF (Backend-For-Frontend)** API in `common-front-api` receives these requests.
- When an investor deposits money, they send it to a special holding account called an **escrow account**.
- An escrow account is a protected bank account where customer funds sit safely separate from company operating cash.
- The controller `WithdrawController.php` manages withdrawal requests and checks balances.
- The controller `UserPaymentController.php` receives bank transfer notifications from NTT Data.
- Every financial movement is immediately saved into database tables named `ECO_TRADE_HIST` and `PST_ALL_TRADE_HIST`.

### Step 2: Batch Scheduling & Invocation Trigger (`st-cli` / Airflow)
- We do not generate heavy PDF reports while a user is clicking on a website.
- Instead, we run a **batch job**, which is a computer script that runs automatically in the background without human intervention.
- A job scheduler called **Apache Airflow** triggers the batch during the night inside a Docker container.
- For local testing, engineers trigger these jobs using a script named `bulk_exec_batch.sh`.
- The main batch files in `st-cli/Report/run/` are `pdf003.php`, `pdf004.php`, and `pdf005.php`.
- Each script receives command-line parameters telling it whether to run for individual customers or for the fund operator, and whether to run daily or monthly.

### Step 3: Pre-Execution Validation & Calendar Check (`st-cli`)
- Before doing heavy calculations, the script runs safety checks.
- It checks the target date and logs a `START` message in our execution history table.
- It checks the business calendar table named `PST_BUSINESS_CALENDAR` to see if Japanese banks were open.
- If today is a weekend or public holiday, the script writes `OK(CLOSING)` to the log and quietly stops with exit code 0.
- For monthly ledgers, it verifies whether today is truly the last calendar day of the month.
- It also loads system configuration settings from `ESYS_CONFIG` to know the maximum number of pages allowed per PDF.

### Step 4: Data Retrieval & Statutory Processing (`st-cli`)
- Once validations pass, the batch queries the database for all confirmed transactions.
- For RT003, `Pdf003Model.php` reads cash balances and trade movements for up to 1,000 customers at a time.
- For RT004, `Pdf004Model.php` collects investment fund share balances and contract histories.
- For RT005, `Pdf005Model.php` aggregates fund-level trades, dividend payouts, and investor capital refunds.
- The script calculates starting rollover balances, adds incoming money, subtracts outgoing fees, and verifies final balances.

### Step 5: PDF/Report Generation & Formatting (`st-cli`)
- The system must turn raw numbers into neat, readable accounting documents.
- It writes exact layout instructions into a temporary text file with page boundaries and coordinate boxes.
- It uses pre-designed visual templates named `RT003.pdf.tpl`, `RT004.pdf.tpl`, and `RT005.pdf.tpl`.
- It executes an internal PDF compilation engine program named `PDF_MAKER_EXEC` (FastPDFGen).
- If a document has too many customers or exceeds page limits, the script cleanly cuts it into multiple parts (`_001.pdf`, `_002.pdf`).

### Step 6: S3 Archival, Directory Publishing & Cleanup (`st-cli`)
- Once the PDF file is successfully compiled on the disk, it must be permanently saved.
- The helper function `upload2S3ForAdmin` in `pdf.php` uploads the file to Amazon S3 cloud storage.
- The script also moves files to organized local folders named `Report/manage/` and `Report/customer/`.
- The batch deletes temporary working text files and scratch folders to keep the server clean.
- Finally, the batch updates the database log to mark the job as `OK` or `NG` (No Good / Error).

### Step 7: Backoffice Access & Electronic Delivery (`st-admin` / `common-front-api`)
- Company backoffice administrators log into our management portal hosted by `st-admin`.
- Nginx web server rules allow administrators to view and download reports through `/report_manage/` and `/report_customer/`.
- At the same time, end-user investors can see their own official disclosures through `ElectronicController.php` in `common-front-api`.
- When an investor opens and acknowledges their report, the system marks the document as read and confirmed.

---

## 4. Meet the Three Statutory Ledgers: RT003, RT004, and RT005

| Ledger Code | Common Name | Real-World Analogy | Who & What It Tracks | Main Script & Template |
| :--- | :--- | :--- | :--- | :--- |
| **RT003** | Cash Customer Ledger (金銭顧客勘定元帳) | A personal bank passbook showing cash in and cash out. | Tracks cash yen balances, deposits, withdrawals, and escrow transfers for individual customers and operators. | `pdf003.php`<br/>`RT003.pdf.tpl` |
| **RT004** | Merchandise Customer Ledger (商品顧客勘定元帳) | An investment portfolio statement showing how many shares of stock you own. | Tracks how many security token units (口数) each customer bought or sold in each specific investment fund. | `pdf004.php`<br/>`RT004.pdf.tpl` |
| **RT005** | Trading Product Ledger (トレーディング商品勘定元帳) | A warehouse inventory register for a store owner. | Tracks the total units, buying prices, distributions, and redemptions of an entire investment product. | `pdf005.php`<br/>`RT005.pdf.tpl` |

---

## 5. Type B — The Distributed Ledger (On-Chain GoQuorum)

Now let us look briefly at the other kind of ledger in OwnersBook+: the **Distributed Ledger**.

### 1. What is On-Chain Data?
- **On-chain** means data that is permanently written onto blocks in a blockchain network.
- We use a private blockchain called **GoQuorum**, which is based on Ethereum.
- When an investment fund is issued, a digital contract creates security tokens following the **ERC-1400** standard.
- When investors trade shares, the tokens are moved between blockchain addresses.

### 2. How We Read On-Chain Data
- Blockchain nodes are not convenient for showing quick web pages.
- A background worker reads blockchain blocks and copies transaction summaries into regular database tables like `bc_blocks`, `bc_tokens`, and `block_tnx`.
- We created a dedicated web viewer application called `st-bc-explorer`.
- Developers and compliance staff visit `st-bc-explorer` to search for transaction hashes, see token holders, and inspect wallet balances.

### 3. How Type B Relates to Type A
- Type A handles legal accounting in Japanese Yen stored in traditional database tables.
- Type B handles digital security tokens stored on the blockchain.
- They are two sides of the same coin, but they use completely separate software pipelines.
- Later verification checks make sure that the tokens on the blockchain match the money recorded in the statutory ledgers.

---

## 6. Comprehensive API Reference Table

Here are all the web endpoints that make this entire ledger ecosystem work across our repositories:

| Method | Endpoint | Repo | Purpose | Main Request | Main Response | Auth | Main Error Codes |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `GET` | `/withdraw` | `common-front-api` | Show the user their registered bank account and current cash balance | None (Header: Bearer Token) | User bank info and withdrawable cash balance | User Token | `E103-0001` (Please log in) |
| `POST` | `/withdraw/confirm` | `common-front-api` | Double-check a withdrawal amount and calculate any bank transfer fees | Requested amount, password, device ID | Fee details and a one-time approval token | User Token | `E103-0504` (Missing info), `E103-0002` (Wrong password) |
| `POST` | `/withdraw/execute` | `common-front-api` | Submit the final cash withdrawal request to our queue | Amount, fees, scheduled date, SMS code | Simple success confirmation | User Token | `E103-0504` (Missing info), `E103-0003` (Not enough money) |
| `POST` | `/withdraw/cancel` | `common-front-api` | Cancel a withdrawal request before it is processed | Request ID number and password | Simple success confirmation | User Token | `E103-0002` (Wrong password) |
| `GET` | `/payment/payin_bank_account` | `common-front-api` | Get the investor's dedicated virtual bank account number for deposits | None (Header: Bearer Token) | Bank code, branch, and account number | User Token | `E103-0001` (Please log in) |
| `GET` | `/payments` | `common-front-api` | Get a list of supported external payment banks | None | List of available payment methods | User Token | `undetermined` |
| `GET` | `/payment/{PAYMENT_ID}` | `common-front-api` | Get connection settings for a selected payment bank | Bank payment ID in the URL | Technical configuration for the bank | User Token | `undetermined` |
| `POST` | `/payment/nttd/bind` | `common-front-api` | Bank webhook confirming an investor sent money via wire transfer | Bank notification payload | Success response to the bank | IP Whitelist / Webhook Secret | `undetermined` |
| `GET` | `/payment/activities` | `common-front-api` | View a history feed of recent deposit and balance activities | None | List of past deposit activities | User Token | `undetermined` |
| `GET` | `/electronic_delivery` | `common-front-api` | List official PDF reports available for the user to view | Filter parameters (date, read status) | List of reports with download links | User Token | `E103-0001` (Please log in) |
| `POST` | `/electronic_delivery/confirm` | `common-front-api` | Save investor agreement to receive reports electronically | Document ID and version number | Simple success confirmation | User Token | `E103-0504` (Missing info), `E103-0278` (Invalid number) |
| `POST` | `/electronic_delivery/read` | `common-front-api` | Mark an electronic statutory report as opened and read | Document ID number | Document details and S3 storage path | User Token | `E103-0539` (Document not found) |
| `GET` | `/electronic_delivery/required_electronic` | `common-front-api` | Check if there are unread documents requiring immediate attention | None | Yes/No flag and required document list | User Token | `undetermined` |
| `GET` | `/report_manage/` | `st-admin` | View and download internal statutory ledger files (RT004, RT005) | Folder browsing request | Web page directory list of downloadable PDFs | Admin Login | `403 Forbidden`, `404 Not Found` |
| `GET` | `/report_customer/` | `st-admin` | View and download customer statutory ledger files (RT003) | Folder browsing request | Web page directory list of downloadable PDFs | Admin Login | `403 Forbidden`, `404 Not Found` |
| `GET` | `/home` | `st-bc-explorer` | Open the main dashboard showing blockchain activity | None | Web page with block and token statistics | Admin / Staff Login | `undetermined` |
| `GET` | `/token/{address}/erc1400_tnx_list` | `st-bc-explorer` | Show all transactions for a specific security token contract | Token contract address in the URL | Web page listing token mints and transfers | Admin / Staff Login | `undetermined` |
| `GET` | `/token/{address}/owner_list` | `st-bc-explorer` | Show which wallet addresses hold units of a token | Token contract address in the URL | Web page listing token holders and amounts | Admin / Staff Login | `undetermined` |
| `GET` | `/block/{block_number}/tnx_list` | `st-bc-explorer` | Show transactions contained in a specific blockchain block | Block number in the URL | Web page showing block details | Admin / Staff Login | `undetermined` |
| `GET` | `/wallet/{wallet_address}/...` | `st-bc-explorer` | Show all tokens owned by a specific digital wallet | Wallet address in the URL | Web page with wallet token balances | Admin / Staff Login | `undetermined` |
| `GET` | `/tnx/{tnx_hash}` | `st-bc-explorer` | Look up a raw cryptographic transaction receipt | Transaction hash in the URL | Web page showing gas, sender, and receiver | Admin / Staff Login | `undetermined` |
| `POST` | `/search` | `st-bc-explorer` | Search the blockchain for any block, transaction, or address | Search text entered in the search bar | Automatically jumps to the matching page | Admin / Staff Login | `undetermined` |

---

## 7. Points to Note & Common Gotchas

1. **Quiet Success on Holidays**:
   - If the batch runs on a Saturday, Sunday, or national holiday, it outputs `OK(CLOSING)` and finishes without making any PDF files. This is intended behavior and not an error.
2. **Monthly Batches Only Run on Month-End**:
   - The monthly trading ledger (RT005) will immediately exit with `OK(CLOSING)` if you run it on any day other than the very last day of the month.
3. **Database Configuration Required**:
   - The RT003 batch will crash if page threshold numbers are missing from the configuration table `ESYS_CONFIG`.
4. **Binary Permissions in Docker Containers**:
   - The PDF generator is a compiled binary executable. If file execution permissions are missing, the generator fails and produces empty files.
5. **Temporary Work Files Disappear**:
   - The batch automatically cleans up its local temporary folders as soon as files are uploaded to AWS S3. If you are looking for local files after a successful run, they will already have been erased.
