# OwnersBook+ Ledger Creation Flow — Visual Guide

This document provides visual diagrams and architectural flowcharts for the ledger creation process in the OwnersBook+ project.

---

## 1. System Architecture Map (Dual Ledger System)

```mermaid
flowchart TB
    classDef front fill:#e1f5fe,stroke:#0288d1,stroke-width:2px;
    classDef db fill:#fff3e0,stroke:#f57c00,stroke-width:2px;
    classDef batch fill:#e8f5e9,stroke:#388e3c,stroke-width:2px;
    classDef storage fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px;
    classDef bc fill:#ede7f6,stroke:#512da8,stroke-width:2px;

    subgraph FrontLayer["1. Ingestion Layer (Front-End & External Gateways)"]
        User["Investor / Web Browser"]:::front
        NTTD["NTT Data / Bank Gateway"]:::front
        FrontAPI["common-front-api<br/>(Laravel BFF)"]:::front
        User -->|/payment/*, /withdraw/*| FrontAPI
        NTTD -->|/payment/nttd/bind| FrontAPI
    end

    subgraph StorageLayer["2. Relational Datastore (Single Source of Truth)"]
        DB[("MySQL Database<br/>(hhd / hhd_pfund)")]:::db
        FrontAPI -->|Inserts cash/trade journal entries| DB
    end

    subgraph BatchLayer["3. Type A: Statutory Batch Processing Engine (st-cli)"]
        Airflow["Apache Airflow 2<br/>(Scheduler & Celery Queue)"]:::batch
        Worker["st-airflow2-batch-daybatch-pfund<br/>(PHP 7.4 CLI Worker)"]:::batch
        BatchRunner["st-cli Report Batch Scripts<br/>(pdf003.php / pdf004.php / pdf005.php)"]:::batch
        PDFEngine["FastPDFGen Binary<br/>(PDF_MAKER_EXEC + *.pdf.tpl)"]:::batch

        Airflow -->|Trigger cron/schedule| Worker
        Worker -->|Executes CLI| BatchRunner
        BatchRunner <-->|Read balances & journal records| DB
        BatchRunner -->|Write .txt script commands| PDFEngine
        PDFEngine -->|Compile binary| BatchRunner
    end

    subgraph OutputLayer["4. Storage, Archival & Publishing"]
        S3[("AWS S3 Bucket<br/>(Permanent Archival)")]:::storage
        LocalDir["Local Mount Folders<br/>(/Report/manage & /Report/customer)"]:::storage
        AdminPortal["st-admin<br/>(Nginx /report_manage/ autoindex)"]:::storage
        UserDelivery["common-front-api<br/>(/electronic_delivery/*)"]:::front

        BatchRunner -->|upload2S3ForAdmin| S3
        BatchRunner -->|Move generated PDF| LocalDir
        LocalDir -->|Static file serving| AdminPortal
        S3 -->|Read-only signed links| UserDelivery
    end

    subgraph TypeBLayer["5. Type B: Distributed Ledger (GoQuorum On-Chain)"]
        Quorum["GoQuorum Blockchain<br/>(ERC-1400 Security Tokens)"]:::bc
        SyncWorker["Blockchain Sync Worker"]:::bc
        BCDB[("MySQL bc_* Cache Tables<br/>(bc_blocks, bc_tokens, block_tnx)")]:::db
        Explorer["st-bc-explorer<br/>(Laravel Explorer Web UI)"]:::bc

        Quorum -->|Event Logs: TransferByPartition| SyncWorker
        SyncWorker -->|Index state| BCDB
        BCDB -->|Query transactions| Explorer
    end
```

---

## 2. End-to-End Sequence Diagram (Type A Statutory Ledger)

```mermaid
sequenceDiagram
    autonumber
    actor Investor as Investor / Operator
    participant API as common-front-api
    participant DB as MySQL (hhd / hhd_pfund)
    participant Airflow as Airflow Scheduler
    participant Batch as st-cli (pdf003/004/005)
    participant Engine as FastPDFGen (PDF_MAKER_EXEC)
    participant S3 as AWS S3 Storage
    participant Admin as st-admin (Backoffice)

    %% Step 1
    rect rgb(235, 245, 255)
    Note over Investor,DB: Step 1: Transaction Initiation & Escrow Recording
    Investor->>API: Deposit money or submit withdrawal request
    API->>DB: Write ECO_TRADE_HIST, PST_ALL_TRADE_HIST, WITHDRAW_APPLY
    end

    %% Step 2 & 3
    rect rgb(255, 250, 235)
    Note over Airflow,DB: Step 2 & 3: Batch Trigger & Safety Validation
    Airflow->>Batch: Run batch (e.g., php pdf003.php 1)
    Batch->>DB: Check business calendar (PST_BUSINESS_CALENDAR)
    alt Holiday or Non-Business Day
        Batch->>DB: Save log = 'OK(CLOSING)'
        Batch-->>Airflow: Exit 0 (Graceful skip)
    else Business Day Open
        Batch->>DB: Save log = 'START'
        Batch->>DB: Read threshold configs from ESYS_CONFIG
    end
    end

    %% Step 4 & 5
    rect rgb(240, 255, 240)
    Note over Batch,Engine: Step 4 & 5: Data Extraction & PDF Rendering
    Batch->>DB: Query customer chunks, rollover balances & trade movements
    Batch->>Batch: Format page rows & calculate running balances
    Batch->>Batch: Create temporary .txt command script
    Batch->>Engine: Execute PDF_MAKER_EXEC with template (*.pdf.tpl)
    Engine-->>Batch: Generate compiled .pdf file (or error in .err log)
    end

    %% Step 6 & 7
    rect rgb(255, 240, 245)
    Note over Batch,Admin: Step 6 & 7: S3 Archival, Directory Publishing & Access
    Batch->>S3: Upload PDF via upload2S3ForAdmin()
    Batch->>Batch: Move PDF to /Report/manage/ or /customer/
    Batch->>Batch: Remove local temporary directory
    Batch->>DB: Save log = 'OK'
    Admin->>Investor: Backoffice staff download via /report_manage/
    API->>Investor: Customer confirms copy via /electronic_delivery/*
    end
```

---

## 3. Step-by-Step Batch Execution Logic & Decision Tree

```mermaid
flowchart TD
    Start([1. Airflow / CLI Invocation]) --> ParseArgs[Parse CLI Arguments: processType, baseDate]
    ParseArgs --> FetchBaseDate[Get base date: getBaseD]
    FetchBaseDate --> LogStart[Write DB Log: 'START']
    LogStart --> CheckCal{Is today a business day?<br/>getCalendarCnt > 0}

    CheckCal -- No (Holiday) --> LogClosing[Write DB Log: 'OK(CLOSING)']
    LogClosing --> Exit0([Exit 0 - Skip without error])

    CheckCal -- Yes --> CheckMonthly{Is batch Monthly mode?<br/>(RT005 kbn=2)}
    CheckMonthly -- Yes --> CheckMonthEnd{Is base date month-end?<br/>checkDateByType == true}
    CheckMonthEnd -- No --> LogClosing
    CheckMonthEnd -- Yes --> LoadConfig[Fetch threshold configs from ESYS_CONFIG]
    CheckMonthly -- No (Daily) --> LoadConfig

    LoadConfig --> ValidateThreshold{Thresholds defined in DB?}
    ValidateThreshold -- Missing --> ThrowErr[Throw Exception: Threshold missing]
    ThrowErr --> LogNG[Write DB Log: 'NG']
    LogNG --> Exit1([Exit 1 - Abort])

    ValidateThreshold -- Valid --> InitDirs[Create temp output directory: makePdfDir]
    InitDirs --> FetchChunk[Fetch customer / fund chunk from DB]

    FetchChunk --> CheckData{Records found?}
    CheckData -- No (0 records) --> LogNoData[Write DB Log: 'OK(NO_DATA)']
    LogNoData --> CleanDir[Remove temporary directory]
    CleanDir --> Exit0

    CheckData -- Yes --> CalcBalances[Compute rollover balance, debits, credits & running totals]
    CalcBalances --> BuildTxt[Build FastPDFGen commands: startPage, drawBox, drawStr]
    BuildTxt --> CheckSplit{Customer count >= limit OR<br/>Pages >= maxPagePerPdf?}
    CheckSplit -- Yes --> CutFile[Append 'finish' & start new chunk _002.pdf]
    CheckSplit -- No --> LoopRows{More records in chunk?}
    LoopRows -- Yes --> CalcBalances
    LoopRows -- No --> SaveTxt[Write commands to .txt file]

    CutFile --> SaveTxt
    SaveTxt --> RunPDFGen[Execute PDF_MAKER_EXEC binary]
    RunPDFGen --> VerifyGen{PDF exists AND<br/>errlog size == 0?}

    VerifyGen -- Failed --> RecordErr[Collect error message from .err log]
    RecordErr --> LogNG

    VerifyGen -- Succeeded --> UploadS3[Upload to S3: upload2S3ForAdmin]
    UploadS3 --> MoveLocal[Move file to Report/manage or Report/customer]
    MoveLocal --> CleanTemp[Remove temporary working folder]
    CleanTemp --> LogOK[Write DB Log: 'OK']
    LogOK --> ExitSuccess([Exit 0 - Completed])
```

---

## 4. Ledger Type Pipeline Comparison (RT003 vs. RT004 vs. RT005)

```mermaid
flowchart LR
    subgraph RT003_Flow["RT003: Cash Customer Ledger (金銭顧客勘定元帳)"]
        direction TB
        DB_RT003[("ECO_USER<br/>ECO_TRADE_HIST<br/>PST_ALL_TRADE_HIST")] --> Model_003["Pdf003Model.php<br/>- Chunks by 1,000 customers<br/>- Process 1 (Customer) or 2 (GP)<br/>- Cash balances & movements"]
        Model_003 --> Tpl_003["RT003.pdf.tpl<br/>(FastPDFGen layout)"]
        Tpl_003 --> Out_003["YYYY-MM-DD_顧客勘定元帳_001.pdf<br/>(Splits if pages > maxPagePerPdf)"]
    end

    subgraph RT004_Flow["RT004: Merchandise Customer Ledger (商品顧客勘定元帳)"]
        direction TB
        DB_RT004[("ECO_USER<br/>PST_BUY_TRADE_HIST<br/>PST_FUND, PST_CLIENT")] --> Model_004["Pdf004Model.php<br/>- Groups by (CLIENT_ID, FUND_CD)<br/>- Tracks token units (口数)<br/>- Acquisition cost & trade price"]
        Model_004 --> Tpl_004["RT004.pdf.tpl<br/>(30 rows per page)"]
        Tpl_004 --> Out_004["YYYY-MM-DD_商品顧客勘定元帳.pdf<br/>(Saved to Report/manage/ymd/)"]
    end

    subgraph RT005_Flow["RT005: Trading Product Ledger (トレーディング商品勘定元帳)"]
        direction TB
        DB_RT005[("PST_FUND<br/>PST_BUY_TRADE_HIST<br/>PST_DIVIDEND_HIST")] --> Model_005["Pdf005Model.php<br/>- Aggregated per fund<br/>- Mode 1: Daily position<br/>- Mode 2: Month-end position<br/>- Dividends & redemptions"]
        Model_005 --> Tpl_005["RT005.pdf.tpl<br/>(30 rows per page)"]
        Tpl_005 --> Out_005["YYYY-MM-DD_トレーディング商品勘定元帳.pdf<br/>(+ '（月次）' if monthly mode)"]
    end
```

---

## 5. Database Journal Entity-Relationship Diagram

```mermaid
erDiagram
    ECO_USER ||--o{ ECO_TRADE_HIST : "owns cash entries"
    ECO_USER ||--o{ PST_BUY_TRADE_HIST : "holds trade contracts"
    PST_FUND ||--o{ PST_BUY_TRADE_HIST : "issued under"
    ECO_TRADE_HIST ||--o| PST_ALL_TRADE_HIST : "links via GUID = TRADE_ID"
    PST_ALL_TRADE_HIST ||--o| PST_BUY_TRADE_HIST : "links via TRADE_NUM"
    PST_FUND ||--o{ PST_DIVIDEND_HIST : "distributes"
    PST_BUSINESS_CALENDAR ||--o{ PSYS_BATCH_EXECUTE_LOG : "governs execution date"

    ECO_USER {
        int SEQ_NO PK
        string SECURITY_ACCOUNT_NUMBER "Client ID"
        int USER_TYPE "1=Individual, 3=Operator/GP, 4=Corporate"
        string NAME
    }

    ECO_TRADE_HIST {
        int GUID PK
        int USER_SEQ_NO FK
        date SETTLE_BOOK_D "Settlement Date"
        int SUMMARY_TYPE "301=Offer, 306=Redeem, 307=Cancel"
        decimal AMOUNT "Cash Amount"
        int CASH_IO_FLG "1=Cash movement"
    }

    PST_ALL_TRADE_HIST {
        string TRADE_ID PK
        string TRADE_NUM FK
        string CLIENT_ID
        int BUYSELL "1=Buy, 2=Sell"
        decimal TRADE_AMOUNT
    }

    PST_BUY_TRADE_HIST {
        string TRADE_NUM PK
        string FUND_CD FK
        string CLIENT_ID
        decimal EXECUTE_QTY "Units (口数)"
        decimal PRICE "Unit Price"
        int TRADE_STATUS "20=Executed, 21=Cancelled"
    }

    PST_FUND {
        string FUND_CD PK
        string FUND_NM
        string GENERAL_PARTNER_ID "Operator Account"
        int FUND_TYPE "1=Physical, 99=Other"
    }

    PST_BUSINESS_CALENDAR {
        date BASE_D PK
        int DATE_TYPE "0=Holiday, 1=Business Day"
        int MONTH_END_TYPE "1,2,3,4=Valid settlement"
    }

    PSYS_BATCH_EXECUTE_LOG {
        int LOG_ID PK
        string BATCH_ID "RT003_1, RT004, RT005_1"
        string BATCH_NAME
        string STATUS "START, OK, OK(CLOSING), OK(NO_DATA), NG"
        date EXECUTE_DATE
    }
```

---

## 6. Type B Distributed Ledger & Explorer Data Flow

```mermaid
flowchart TD
    subgraph NodeLayer["GoQuorum Private Blockchain"]
        Contract["ERC-1400 Smart Contract<br/>(Security Token Master)"]
        Blocks["GoQuorum Blocks<br/>(Private EVM Consensus)"]
        Contract -->|Mints / Transfers / Burns| Blocks
    end

    subgraph SyncLayer["Off-Chain Synchronization Service"]
        RPC["GoQuorum RPC (:22000)"]
        SyncWorker["Indexer Process"]
        Blocks --> RPC
        RPC -->|JSON-RPC Poll / Event Stream| SyncWorker
    end

    subgraph CacheDB["Off-Chain Caching Relational Database"]
        bc_blocks["bc_blocks<br/>(Block number, hash, timestamp)"]
        bc_tokens["bc_tokens<br/>(Contract address, symbol, total supply)"]
        block_tnx["block_tnx<br/>(Tx hash, partition, sender, receiver, amount)"]
        wallet_balance["wallet_balance<br/>(Wallet address, token address, balance)"]

        SyncWorker -->|Write blocks| bc_blocks
        SyncWorker -->|Write tokens| bc_tokens
        SyncWorker -->|Write transactions| block_tnx
        SyncWorker -->|Update balances| wallet_balance
    end

    subgraph ViewerLayer["st-bc-explorer (Laravel Web Explorer)"]
        HomeController["HomeController<br/>- Dashboard metrics (/home)<br/>- Search (/search)"]
        TokenController["TokenController<br/>- Token txs (/token/{addr}/erc1400_tnx_list)<br/>- Holders (/token/{addr}/owner_list)"]
        BlockController["BlockController<br/>- Block txs (/block/{num}/tnx_list)"]
        WalletController["WalletController<br/>- Balances (/wallet/{addr}/...)"]
        TnxController["TnxController<br/>- Tx receipt (/tnx/{hash})"]

        bc_blocks --> BlockController
        bc_tokens --> TokenController
        block_tnx --> TnxController
        wallet_balance --> WalletController
        bc_blocks & bc_tokens & block_tnx --> HomeController
    end
```

---

## 7. Component Responsibility Matrix

| Stage | Responsibility | Primary Repository | Key Files & Classes | Output Artifact |
| :--- | :--- | :--- | :--- | :--- |
| **1. Ingestion** | Accept fiat deposit / withdrawal requests and record ledger entries | `common-front-api` | `WithdrawController.php`<br/>`UserPaymentController.php` | Rows in `ECO_TRADE_HIST` & `WITHDRAW_APPLY` |
| **2. Orchestration** | Trigger batch containers according to trading calendar | `local-docker`<br/>`dc-airflow2` | `bulk_exec_batch.sh`<br/>`st-airflow2-scheduler` | Container command: `php pdf003.php 1` |
| **3. Safety Check** | Confirm business open status and load page limits | `st-cli` | `Pdf003Model::getCalendarCnt()`<br/>`ConfigModel::get()` | `PSYS_BATCH_EXECUTE_LOG` (Status = `START`) |
| **4. Calculation** | Fetch customer records, rollover balances & calculate net changes | `st-cli` | `Pdf003Model.php`<br/>`Pdf004Model.php`<br/>`Pdf005Model.php` | Structured memory arrays of balanced transactions |
| **5. PDF Assembly** | Render page frames, fonts, and box coordinates | `st-cli` | `FastPDFGen` binary (`PDF_MAKER_EXEC`)<br/>`RT003.pdf.tpl`, `RT004.pdf.tpl`, `RT005.pdf.tpl` | Compiled PDF file (e.g. `YYYY-MM-DD_顧客勘定元帳_001.pdf`) |
| **6. Archival** | Upload to cloud and delete local temp working directories | `st-cli` | `pdf.php` (`upload2S3ForAdmin`)<br/>`remove_directory()` | Permanent S3 object + published local directory |
| **7. Distribution** | Provide secure downloads to backoffice and investors | `st-admin`<br/>`common-front-api` | `pfund-admin.conf` (`/report_manage/`)<br/>`ElectronicController.php` | Admin browser download & Investor app disclosure |
| **Type B Viewer** | Inspect on-chain security token balances and blocks | `st-bc-explorer` | `HomeController.php`<br/>`TokenController.php`<br/>`routes/web.php` | HTML web view of distributed ledger transactions |
