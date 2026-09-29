# Yêu cầu thiết lập môi trường cho các dự án (local-docker)

Tài liệu này tổng hợp thông tin cần thiết để khởi chạy 8 dự án local, dựa trên môi trường docker được cung cấp tại `local-docker/`.

## 0. Bảng ánh xạ dự án ↔ repo ↔ service

| # | Dự án | Repo | Runtime | Container (php-fpm / node) | Vhost nginx | URL |
|---|---|---|---|---|---|---|
| 1 | Fund Management Dashboard | `st-admin` | PHP 7.4 (`st/php:7.4-fpm`) | `st-app-pfund-admin` | `pfund-admin.conf` | http://local-st-admin-hsec.crudist.tech/ |
| 2 | Company-specific Management Dashboard | `company-admin` | PHP 7.4 | `st-app-cmpadmin` | `company-admin.conf` | http://local-hhd-admin-hsec.crudist.tech/ |
| 3 | Common Management Dashboard (BasisAdmin) | `basis-admin` | PHP 7.4 | `st-app-basadmin` | `basis-admin.conf` | http://local-basis-admin-hsec.crudist.tech/ |
| 4 | Common API | `common-front-api` | PHP 8.3.16 | `st-app` (dùng chung với #5) | `common-front-api.conf` | http://local-common-front-api-hsec.crudist.tech/ |
| 5 | Authentication API | `passport` | PHP 8.3.16 | `st-app` (dùng chung với #4) | `passport.conf` | http://local-passport-api-hsec.crudist.tech/ |
| 6 | Fund API | `st-api` | PHP 8.3.16 | `st-app-pfund` | `pfund-api.conf` | http://local-st-front-api-hsec.crudist.tech/ |
| 7 | Trading Site | `app-vuejs` | Node 18 + webpack-dev-server | `front-app-dev` + proxy `app-vuejs-nginx` (:8080) | stack `dc-app-vuejs-dev` | http://local-st-front-hsec.crudist.tech:8080/ |
| 8 | Account Opening Site | `mobile-registration` | Node 18 + webpack-dev-server | `front-reg-dev` + proxy `front-reg-dev-proxy` (:8081) | stack `dc-reg-vuejs-dev` | http://local-register-hsec.crudist.tech:8081/ |

Lưu ý: dịch vụ #4 và #5 chạy chung một container php-fpm `st-app`, chỉ khác nhau ở `root`/`fastcgi_param` theo từng vhost nginx — restart/rebuild container này ảnh hưởng tới cả hai.

---

## 1. Các môi trường Docker cần thiết cho từng repo

### 1.1 Hạ tầng dùng chung (bắt buộc cho mọi dự án)
- Docker network: `st_net` (tạo qua `make start-network`, mọi stack đều khai báo `external: true`)
- Stack `dc-base`: MySQL (`st/mysql:8.0.40`), Redis (`st/redis:7.2`), Postfix, MinIO (+ job `createbuckets` khởi tạo bucket)
- Stack `dc-web`: nginx dùng chung `st-web` (`st/nginx:1.27`) + tất cả container PHP-FPM

### 1.2 Base images cần build trước (qua `make all`/`make build` trong từng thư mục `docker-*`)
- `docker--php74`, `docker-php74-fpm` → image `st/php:7.4-fpm` (dùng cho #1, #2, #3)
- `docker--php83`, `docker-php83-fpm` → image `st/php:8.3-fpm` (dùng cho #4, #5, #6). Đính chính: yêu cầu ban đầu ghi các dự án này là "@php8.4" — thông tin đó không chính xác. Theo đúng Dockerfile:
  - `docker--php83/Makefile` clone `docker-library/php` ở commit `cb522cdc4da1143b550aac8aaaf9306fa24a1f8a` (comment trong Makefile: "Update 8.3 to 8.3.16"), build từ `github/8.3/bookworm/fpm/Dockerfile` → tag `base/php:8.3-fpm`.
  - `docker-php83-fpm/Dockerfile` build `FROM base/php:8.3-fpm`, cài thêm extension (bcmath, zip, pdo_mysql, gd, opcache, mysqli, redis, msgpack) → tag cuối `st/php:8.3-fpm`.
  - Đã xác nhận runtime thực tế bằng `php -v` trong container: **PHP 8.3.16**, không phải 8.4.
- `docker--centos8-php74-cli`, `docker--centos8-php83-cli` → image CLI (composer, artisan, phpcs...)
- `docker-nginx` → image `st/nginx:1.27` (base cho vhost nginx `st-web`)
- `docker-mysql` → image `st/mysql:8.0.40`
- `docker-redis` → image `st/redis:7.2`
- `docker-node` → image `st/node:18` (dùng khi cần base node riêng)
- `docker-openresty` → image openresty (dùng làm reverse-proxy cho 2 app Vue)
- Không cần: `docker-airflow*`, `docker-blockchain`, `docker-postgres`, `docker-postfix`, `docker-openssl-rpm` (phục vụ batch/blockchain, ngoài phạm vi 8 dự án)

### 1.3 Container/service cụ thể theo repo

| Repo | Container chạy trong `dc-web` | Composer/package | File env cần copy |
|---|---|---|---|
| `st-admin` | `st-app-pfund-admin` (image `st/php:7.4-fpm`) | `composer.local.json` (`composer-with-env-docker.sh`) | không có `.env` riêng, cấu hình qua DB/`ESYS_CONFIG` |
| `company-admin` | `st-app-cmpadmin` (image `st/php:7.4-fpm`) | `composer.local.json` | không có `.env`; DB creds inject qua `fastcgi_param` trong `company-admin.conf` |
| `basis-admin` | `st-app-basadmin` (image `st/php:7.4-fpm`) | `composer.local.json` | `.env.local` → copy thành `.env` |
| `common-front-api` | `st-app` (image `st/php:8.3-fpm`) | `composer.local.json` | `.env.local` → copy thành `.env` |
| `passport` | `st-app` (dùng chung container với `common-front-api`) | `composer.local.json` | `.env.local` → copy thành `.env` |
| `st-api` | `st-app-pfund` (image `st/php:8.3-fpm`) | `composer.local.json` | không có `.env` riêng |
| `app-vuejs` | `front-app-dev` (stack `dc-app-vuejs-dev`, node 18) + proxy `app-vuejs-nginx` | npm | `.env.local` → copy thành `.env` |
| `mobile-registration` | `front-reg-dev` (stack `dc-reg-vuejs-dev`, node 18) + proxy `front-reg-dev-proxy` | npm | không có `.env.local`; cấu hình build-time nằm ở `config/local.env.js` |

### 1.4 Biến môi trường điều phối
- `DOCKER_ENV=local` trong `.env` của `dc-web`, `dc-app-vuejs-dev`, `dc-reg-vuejs-dev` — quyết định dùng bộ config nào trong `local-docker/etc/{DOCKER_ENV}/...` (nginx, php-fpm.d, mysql, postfix, fluent_bit).
- `local-docker/.env` (root): `GITLAB_USER`, `GITLAB_PASS` — dùng để build image PHP-FPM qua build-arg khi clone các thư viện private trong lúc `composer install`. **Cần dùng tài khoản/token GitLab cá nhân của người thiết lập, không tái sử dụng token đã có sẵn.**
- Không có biến chọn version PHP — version PHP cố định theo `image:` khai báo cho từng service trong `dc-web/docker-compose.yml`.
- Không dùng HTTPS/mkcert — toàn bộ vhost đều `listen 80;` (HTTP thuần).

---

## 2. Các bước thiết lập đường dẫn để truy cập từ trình duyệt

### 2.1 Cấu hình `/etc/hosts` (trỏ về 127.0.0.1)
```
127.0.0.1 local-st-admin-hsec.crudist.tech
127.0.0.1 local-hhd-admin-hsec.crudist.tech
127.0.0.1 local-basis-admin-hsec.crudist.tech
127.0.0.1 local-common-front-api-hsec.crudist.tech
127.0.0.1 local-passport-api-hsec.crudist.tech
127.0.0.1 local-st-front-api-hsec.crudist.tech
127.0.0.1 local-st-front-hsec.crudist.tech
127.0.0.1 local-register-hsec.crudist.tech
```
(README của `local-docker` còn liệt kê thêm một số host khác phục vụ blockchain/airflow/minio — không cần thiết cho 8 dự án trên.)

### 2.2 Ánh xạ host → service (đã cấu hình sẵn trong `local-docker/etc/local/nginx/*.conf`)
| server_name | root / fastcgi_pass |
|---|---|
| local-st-admin-hsec.crudist.tech | root `/www/st-admin/Webroot`; `fastcgi_pass st-app-pfundadmin:9000` |
| local-hhd-admin-hsec.crudist.tech | root `/www/company-admin/public`; `fastcgi_pass st-app-cmpadmin:9000` |
| local-basis-admin-hsec.crudist.tech | root `/www/basis-admin/public`; `fastcgi_pass st-app-basadmin:9000` |
| local-common-front-api-hsec.crudist.tech | root `/www/common-front-api/public`; `fastcgi_pass st-app:9000` |
| local-passport-api-hsec.crudist.tech | root `/www/passport/public`; `fastcgi_pass st-app:9000` |
| local-st-front-api-hsec.crudist.tech | root `/www/st-api/Webroot`; `fastcgi_pass st-app-pfund:9000` |
| local-st-front-hsec.crudist.tech:8080 | proxy tới `front-app-dev:8080` qua container `app-vuejs-nginx` (stack `dc-app-vuejs-dev`, port publish `8080:80`) |
| local-register-hsec.crudist.tech:8081 | proxy tới `front-reg-dev:8080` qua container `front-reg-dev-proxy` (stack `dc-reg-vuejs-dev`, port publish `8081:8080`) |

Với 2 site Vue, `docker-compose.yml` của webpack-dev-server không publish port trực tiếp; phải qua container proxy (openresty) mới truy cập được từ trình duyệt. Proxy này còn yêu cầu basic-auth (`htpasswd`) và biến Redis (`REDIS_HOST/PORT/DB/TOKEN_MAX_AGE`) để phục vụ cơ chế "click token".

### 2.3 File env cấu hình URL/API cho 2 site Vue
- `app-vuejs/.env.local` (copy thành `.env`) đã trỏ đúng các API local:
  ```
  VUE_APP_FUND_API=http://local-st-front-api-hsec.crudist.tech
  VUE_APP_COMMON_API=http://local-common-front-api-hsec.crudist.tech
  VUE_APP_PASSPORT_API=http://local-passport-api-hsec.crudist.tech
  VUE_APP_REGISTER_URL=http://local-register-hsec.crudist.tech
  ```
  (`.env`/`.env.dev` hiện tại trong repo đang trỏ tới stg/jdata.tk — cần đổi sang `.env.local` để chạy đúng local.)
- `mobile-registration`: không có `.env.local`; giá trị `PASSPORT_API` được hardcode trong `config/local.env.js` (`http://local-passport-api-hsec.crudist.tech`) — đây là nguồn cấu hình build-time thực sự, không phải dotenv.

---

## 3. Lệnh khởi chạy cho từng dự án

### 3.1 Thứ tự khởi chạy chung (một lần, theo README của `local-docker`)
```bash
cd local-docker
make start-network                 # tạo network st_net

# build base images (chạy trong từng thư mục)
cd docker--php74 && make all && cd ..
cd docker-php74-fpm && make all && cd ..
cd docker--centos8-php74-cli && make all && cd ..
cd docker--php83 && make all && cd ..
cd docker-php83-fpm && make all && cd ..
cd docker--centos8-php83-cli && make all && cd ..
cd docker-node && make all && cd ..
cd docker-nginx && make all && cd ..
cd docker-mysql && make all && cd ..
cd docker-redis && make all && cd ..
cd docker-openresty && make all && cd ..

cp env.sample .env    # điền GITLAB_USER / GITLAB_PASS (dùng token cá nhân)
make myphpfpm         # build image myphpfpm dùng cho composer install
```

### 3.2 Hạ tầng nền
```bash
cd dc-base && cp .env.local .env && docker-compose up -d   # MySQL, Redis, Postfix, MinIO
```
Sau đó tạo các DB cần thiết trong `st-mysql` (`basis`, `hhd`, `hhd_mynumber`, `hhd_pfund`, `hhd_stock`) và import dữ liệu (theo README nội bộ, cần quyền truy cập DB nguồn — không thuộc phạm vi tài liệu này).

### 3.3 Dự án #1–#6 (PHP: st-admin, company-admin, basis-admin, common-front-api, passport, st-api)
```bash
cd dc-web
cp .env.local .env
# copy các env riêng theo từng service: .env_st-app, .env_st-app-cmpadmin,
# .env_st-app-pfund, .env_st-migration, .env_st-app-basadmin, ...
docker-compose up -d      # (hoặc dùng: make start-web ở thư mục local-docker gốc)
```
- Cài dependency PHP cho từng service (chạy trong container):
  ```bash
  docker-compose exec st-app-pfund-admin bash
  cd /www/st-admin && COMPOSER=composer.local.json ./composer-with-env-docker.sh install -o -vvv
  ```
  (lặp lại tương tự cho `st-app-cmpadmin` → `/www/company-admin`, `st-app-basadmin` → `/www/basis-admin`, `st-app` → `/www/common-front-api` và `/www/passport`, `st-app-pfund` → `/www/st-api`)
  Có thể dùng `make composer-install-all` ở `local-docker` để chạy composer install cho toàn bộ repo PHP cùng lúc.
- Truy cập trực tiếp qua các URL ở mục 0 (nginx `st-web` phục vụ sẵn khi `dc-web` đã `up`).
- Quản lý: `make restart-web`, `make status-web` (chạy tại thư mục `local-docker`).

### 3.4 Dự án #7 — Trading Site (app-vuejs)
```bash
cd local-docker
make install-app-vuejs      # npm install trong container tạm
make start-app-vuejs        # cd dc-app-vuejs-dev && docker-compose up -d
# hoặc trực tiếp:
cd dc-app-vuejs-dev && cp .env.local .env && make npm_install && docker-compose up -d
```
- Container `front-app-dev` chạy `npm run dev.local` (webpack-dev-server, host `0.0.0.0`, port `8080`).
- Truy cập qua proxy: http://local-st-front-hsec.crudist.tech:8080/
- Log/trạng thái: `make logs-app-vuejs`, `make status-app-vuejs`, `make restart-app-vuejs`.

### 3.5 Dự án #8 — Account Opening Site (mobile-registration)
```bash
cd local-docker
make install-regist-vuejs
make start-regist-vuejs     # cd dc-reg-vuejs-dev && docker-compose up -d
# hoặc trực tiếp:
cd dc-reg-vuejs-dev && cp .env.local .env && make npm_install && docker-compose up -d
```
- Container `front-reg-dev` chạy `npm run local` (webpack-dev-server, host `0.0.0.0`, port `8080` nội bộ).
- Truy cập qua proxy: http://local-register-hsec.crudist.tech:8081/
- Log/trạng thái: `make logs-regist-vuejs`, `make status-regist-vuejs`, `make restart-regist-vuejs`.

---

## 4. Ghi chú / rủi ro cần lưu ý

- `local-docker/.env` hiện đã có sẵn `GITLAB_PASS` là token thật — đây là thông tin nhạy cảm, không nên commit/chia sẻ. Người thiết lập môi trường mới nên dùng token cá nhân riêng.
- Trong `dc-web/docker-compose.yml`, volume mount của `st-migration` dùng cú pháp `{$DOCKER_ENV}` (không phải `${DOCKER_ENV}` chuẩn) — không được biến `docker-compose` thay thế, dẫn tới thư mục `etc/{local}` theo nghĩa đen tồn tại trên đĩa. Đây là điểm cần lưu ý khi debug nếu `st-migration` không đọc đúng config.
- Không có cấu hình mkcert/TLS — toàn bộ URL local đều là `http://`.
- Repo #4 và #5 (`common-front-api`, `passport`) dùng chung một container PHP-FPM (`st-app`) — thao tác restart/rebuild ảnh hưởng cả hai.
- Composer trong các container chạy dưới user `www-data`, thư mục home là `/var/www` (không phải `/root`) — auth.json cho GitLab private repo phải đặt tại `/var/www/.composer/auth.json`.
- `etc/local/nginx/quorum-lb.conf` tham chiếu upstream `quorum-node1:22000` (stack blockchain, không nằm trong 8 dự án) — nếu stack blockchain không được khởi chạy, nginx `st-web` sẽ crash-loop lúc load lại config vì không resolve được host này. Đã đổi tên thành `quorum-lb.conf.disabled` để `st-web` khởi động ổn định khi chỉ chạy 8 dự án mục tiêu (theo đúng pattern đã có sẵn của `airflow.conf.disabled`).
- `st-api`: `composer.lock` khóa `netsec/common-balance` yêu cầu đúng PHP `8.3.12`, trong khi image `st/php:8.3-fpm` là `8.3.16` — phải cài bằng `composer install --ignore-platform-reqs`.
- **[ĐÃ SỬA]** Schema `hhd` (company-admin): migration dùng cột `RANK` không có backtick trong câu lệnh trigger `ON DUPLICATE KEY UPDATE` — `RANK` là từ khóa dành riêng từ MySQL 8.0, gây lỗi cú pháp trên image `st/mysql:8.0.40`. Nguyên nhân gốc nằm ở 2 hàm build SQL trigger dùng chung `MyTriggerSQLTrait::appendColumns()` và `MyTriggerTrait::createTriggerSQL()` (nối tên cột thẳng vào câu SQL không backtick — ảnh hưởng TẤT CẢ trigger tự sinh trong toàn bộ repo, không riêng `hhd`). Đã sửa: bọc backtick quanh mọi tên cột (`` `{col}` = NEW.`{col}` ``) trong cả 2 hàm này.
  - Trong lúc migrate lại cũng gặp thêm 1 bug độc lập: 2 file migration `2023_12_12_090215_add_column_to_eco_user_notice_table.php` và `2024_01_18_100215_add_column_to_eco_user_notice_table.php` cùng có phần tên (sau timestamp) giống hệt nhau → Laravel suy ra cùng 1 tên class `AddColumnToEcoUserNoticeTable` → PHP fatal "Cannot declare class" khi cả 2 cùng pending trong 1 lần chạy. Đã sửa bằng cách **đổi cả tên file lẫn tên class** của file 2024-01-18 thành `add_fund_cd_column_to_eco_user_notice_table.php` / `AddFundCdColumnToEcoUserNoticeTable` (đúng quy ước Laravel: tên class phải suy ra được từ tên file), rồi chạy lại `composer dump-autoload` trong container `st-migration` để xoá entry classmap cũ.
  - Sau 2 fix trên, đã migrate xong **toàn bộ 4 schema**: `hhd` (folder `company`), `hhd_pfund` (folder `company_fund`), `hhd_mynumber` (folder `company_mynumber`), `hhd_stock` (folder `company_stock`) — không còn lỗi nào. Lệnh đã chạy:
    ```bash
    docker exec st-migration sh -c 'cd /www/migration && php artisan migrate --folder company --target-db hhd --force'
    docker exec st-migration sh -c 'cd /www/migration && php artisan migrate --folder company_fund --target-db hhd_pfund --force'
    docker exec st-migration sh -c 'cd /www/migration && php artisan migrate --folder company_mynumber --target-db hhd_mynumber --force'
    docker exec st-migration sh -c 'cd /www/migration && php artisan migrate --folder company_stock --target-db hhd_stock --force'
    ```
  - Thú vị: có 1 migration về sau (`2024_08_06_090000_remove_rank_ECO_USER_table`) tự xóa hẳn cột `RANK` — xác nhận đây là vấn đề tương thích MySQL 8 mà chính dev gốc cũng từng gặp và xử lý ở phiên bản sau.
- 2 site Vue (Trading Site :8080, Account Opening :8081) yêu cầu HTTP basic-auth ở tầng proxy openresty (`htpasswd` chỉ chứa hash, không có plaintext trong repo) — cần cung cấp thêm để truy cập qua trình duyệt.
- **Cập nhật**: repo `migration` có sẵn command `php artisan import:config {esys_config|bsys_config} {dev|testing|stg|prod} {target_db}` để nạp dữ liệu **config** (không phải seed nghiệp vụ) từ file tĩnh `config/bsys_config_dev.php`/`config/esys_config_dev.php` vào bảng `BSYS_CONFIG`/`ESYS_CONFIG`. Đã chạy thử:
  ```bash
  docker exec st-migration sh -c 'cd /www/migration && php artisan import:config bsys_config dev basis'
  ```
  → Nạp thành công **25 dòng** vào `basis.BSYS_CONFIG` (xác nhận bằng `SELECT COUNT(*)`). Lưu ý: file `bsys_config_dev.php` chứa sẵn 1 cặp AWS access key/secret thật — cẩn thận khi chia sẻ file này.
- Tuy nhiên sau khi import, test lại `st-api` và `st-admin` vẫn lỗi y hệt (`Unknown database '_pfund'` / `unknown_error`). Truy vết qua log (`/www/st-api/App/Logs/all.log`) cho thấy nguyên nhân **không phải** do thiếu `BSYS_CONFIG`, mà do truy vấn `SELECT SEQ_NO,DB_NAME FROM basis.BCO_COMPANY WHERE COMPANY_ID = 'sgDj8sHr'` trả về rỗng — bảng `BCO_COMPANY` (ánh xạ company_id → tên DB, dùng bởi cả st-admin lẫn st-api để xác định `{prefix}_pfund`) **hoàn toàn chưa có dữ liệu**.
- Đây là **master/business data thật** (danh sách công ty), không phải config — nằm ngoài phạm vi của `import:config`. Migration repo cũng chủ động không dùng Laravel seeder cho loại dữ liệu này (README ghi rõ mục "Seedingを使わない" — lý do: cấu trúc bảng master hay đổi, seed 1 lần dễ lỗi thời). Vậy việc có dữ liệu `BCO_COMPANY` (và các bảng nghiệp vụ khác) vẫn thực sự cần import từ STG/STG2 qua SSH nội bộ như đã ghi nhận trước đó — `import:config` chỉ giải quyết được phần cấu hình hệ thống (system config), không giải quyết được phần dữ liệu nghiệp vụ.

## [ĐÃ GIẢI QUYẾT] Import dữ liệu STG thật — 2026-09-04

Bạn đã tự tải được file backup (`my-folder/db/stg_db_0904.tar.gz`) — đây chính là bộ dump STG mà README yêu cầu (khớp chính xác nội dung: `basis`, `hhd`, `hhd_mynumber`, `hhd_pfund`, `hhd_stock`). File còn lại `airflow_stg_backup_0904.dump` là **PostgreSQL dump metadata của Airflow, không liên quan tới 8 dự án** nên không dùng.

**Các bước đã thực hiện:**
```bash
# 1. Giải nén
tar -xzf my-folder/db/stg_db_0904.tar.gz -C my-folder/db/extracted
gunzip -k my-folder/db/extracted/dbbk/*.gz

# 2. Copy vào container và import (basis trước, hhd để cuối vì nặng nhất ~331MB)
docker cp <file>.sql st-mysql:/tmp/<db>.sql
docker exec st-mysql sh -c "mysql -h127.0.0.1 -uroot -pQampa4jLoxCSoXsG <db> < /tmp/<db>.sql"
```
Cả 5 database import thành công, không lỗi. Dump là chuẩn `mysqldump` đầy đủ (`DROP TABLE IF EXISTS` + `CREATE TABLE` + `INSERT`) nên **ghi đè hoàn toàn** schema+data cũ (kể cả kết quả migrate thủ công trước đó) bằng đúng snapshot STG ngày 2026-09-04.

**Sự cố gặp phải khi chạy tiếp câu lệnh `UPDATE ESYS_CONFIG/BSYS_CONFIG`** (bước repoint URL/S3 sang local theo README): lúc đầu dùng `docker exec st-mysql mysql ... <<'EOF' ... EOF` (heredoc) — **chạy "thành công" (exit 0) nhưng thực ra không làm gì cả**, vì `docker exec` mặc định KHÔNG forward stdin từ host vào container (thiếu cờ `-i`), nên toàn bộ heredoc rơi vào hư không. Cách sửa đúng: ghi các câu UPDATE ra file `.sql`, `docker cp` vào container, rồi chạy `mysql ... < /tmp/file.sql` bằng `sh -c` (redirect xảy ra bên trong container, không phụ thuộc stdin của `docker exec`). Đây là điểm cần nhớ: **`docker exec` không có `-i` sẽ không nhận input qua heredoc/pipe, luôn im lặng "thành công" — phải verify bằng SELECT lại, không tin exit code.**

**Kết quả sau khi import + repoint:**
| Bảng | Trước | Sau |
|---|---|---|
| `basis.BCO_COMPANY` | 0 dòng | 1 dòng (`sgDj8sHr` → Loadstar Securities, `DB_NAME=hhd`) |
| `hhd.ECO_ADMIN_USER` | 0 dòng | 78 dòng (user thật: `admin`, `zhaot`, `yamada`, `saito`...) |
| `hhd.ECO_USER` | 0 dòng | 251 dòng |
| `hhd.ESYS_CONFIG` | 0 dòng | 227 dòng, đã repoint URL API + S3 sang local/minio |

**Retest sau khi import — kết quả thật, không còn giả lập:**
- ✅ **st-admin** (`admin/123456`): **đăng nhập thành công hoàn toàn**, vào đúng dashboard "匿名組合ファンド管理画面" với đầy đủ menu nghiệp vụ (業務管理, ファンド管理, WEB管理, 照会, 会社管理) + nút logout. Lỗi `_pfund` đã biến mất hoàn toàn.
- ✅ **basis-admin** (`admin/admin1234`): vẫn đăng nhập được bình thường sau khi `basis` bị ghi đè bởi STG (không bị mất tài khoản test).
- ✅ **st-api**: gọi với header `X-Ci: sgDj8sHr` không còn lỗi resolve company nữa — giờ trả `404 router error` khi gọi `/` (bình thường, vì route gốc không được định nghĩa cho API này).
- ⚠️ **company-admin** (`arakawa/1qazxsw2`): user `arakawa` **không tồn tại** trong dữ liệu STG này (đã kiểm tra kỹ). Danh sách user thật hiện có: `admin, zhaot, windward1-4, yamada, csuser1-3, saito, testadmin01, lichaox, uatuser, opeuser1-3, mktuser1-3...`. Có thể `arakawa` chỉ tồn tại ở production, không có trong snapshot STG này.
- ⚠️ **app-vuejs** (`10010000212/a_testing18`): không tìm thấy `USER_ID`/`SECURITY_ACCOUNT_NUMBER` khớp `10010000212` trong `hhd.ECO_USER` — tài khoản test này cũng không có trong STG snapshot. Ngoài ra API login của `passport` (`POST /user/login`) yêu cầu mật khẩu đã **mã hoá** kèm `SECRET_VERSION_ID` nên không thể test bằng plaintext qua curl đơn giản — cần test qua UI thật (vẫn đang bị basic-auth chặn ở proxy `:8080`).

**Kết luận**: gap dữ liệu nghiệp vụ (mục đã ghi ở trên) coi như **đã giải quyết xong** ở tầng hạ tầng/dữ liệu chung (`BCO_COMPANY`, `ESYS_CONFIG`, users...). 2 tài khoản test cụ thể (`arakawa`, `10010000212`) trong yêu cầu ban đầu đơn giản là không có trong snapshot STG ngày 2026-09-04 — đây là vấn đề về **độ mới/phạm vi của bản backup**, không phải lỗi môi trường.

---

## 5. Thông tin kết nối Database

### 5.1 MySQL (`st-mysql`, image `st/mysql:8.0.40`)
| Thông tin | Từ trong container (dùng bởi app) | Từ máy host (dùng để connect bằng client ngoài, vd DBeaver/CLI) |
|---|---|---|
| Host | `st-mysql` | `127.0.0.1` |
| Port | `3306` | `3306` (đã publish `3306:3306` trong `dc-base/docker-compose.yml`) |
| User | `root` | `root` |
| Password | `Qampa4jLoxCSoXsG` | `Qampa4jLoxCSoXsG` |

Lệnh connect nhanh từ host:
```bash
mysql -h127.0.0.1 -P3306 -uroot -pQampa4jLoxCSoXsG
```
Chỉ có 1 user `root@%` — không có user riêng nào khác được tạo (README có hướng dẫn tạo thêm `migration_user` nhưng bước đó nằm trong luồng import dữ liệu STG chưa thực hiện).

**5 database hiện có** (đã tạo, đa số chỉ có schema rỗng — xem mục 4):
| Database | Dùng bởi |
|---|---|
| `basis` | common-front-api, passport, st-api, basis-admin — dữ liệu hệ thống dùng chung (`BSYS_CONFIG`, `BCO_COMPANY`...) |
| `hhd` | company-admin — dữ liệu theo từng công ty |
| `hhd_pfund` | st-admin, st-api — dữ liệu quỹ (fund) theo công ty |
| `hhd_mynumber` | company-admin (module mynumber) |
| `hhd_stock` | company-admin (module stock) |

### 5.2 Redis (`st-redis`)
| Thông tin | Từ trong container | Từ máy host |
|---|---|---|
| Host | `st-redis` | `127.0.0.1` |
| Port | `6379` | `5003` (đã publish `5003:6379` trong `dc-base/docker-compose.yml`) |
| Password | (không có) | (không có) |

### 5.3 MinIO (thay thế S3 cho local)
| Thông tin | Giá trị |
|---|---|
| Endpoint (trong container) | `http://minio:9000` |
| Endpoint (từ host) | `http://127.0.0.1:9000` (API), console `http://127.0.0.1:8900` |
| Root user/pass | `root` / `12345678` |
| App user/pass (dùng trong `ESYS_CONFIG`/`BSYS_CONFIG` để trỏ AWS key) | `minio-user` / `minio-password` |
| Bucket đã tạo sẵn | `basis-bucket`, `hhd-hsec-general`, `hhd-hsec-kyc`, `hhd-hsec-mynumber`, `hhd-hsec-s3-for-gcp-bigquery-datatransfer`, `hhd-st-tax`, `local-hhd-st-digital` |

### 5.4 Chi tiết kết nối DB theo từng dự án

| # | Dự án | Repo | File cấu hình DB | DB_HOST | DB_DATABASE | User/Pass |
|---|---|---|---|---|---|---|
| 1 | Fund Management Dashboard | `st-admin` | env truyền qua `dc-web/.env_st-app-pfund` (không có `.env` riêng trong repo) | `st-mysql` | động: `{prefix}_pfund` (tra từ `basis.BCO_COMPANY`, mặc định test `hhd_pfund`) | root / `Qampa4jLoxCSoXsG` |
| 2 | Company-specific Mgmt | `company-admin` | `fastcgi_param` trong `etc/local/nginx/company-admin.conf` | `st-mysql` | `hhd` (biến `DB_MASTER_NAME`), cộng `basis` (biến `BASIS_DB_NAME`) | root / `Qampa4jLoxCSoXsG` |
| 3 | BasisAdmin | `basis-admin` | `basis-admin/.env` (copy từ `.env.local`) | `st-mysql` | `basis` | root / `Qampa4jLoxCSoXsG` |
| 4 | Common API | `common-front-api` | `common-front-api/.env` (copy từ `.env.local` — **vừa tạo lại, trước đó bị thiếu hoàn toàn**) | `st-mysql` | `basis` | root / `Qampa4jLoxCSoXsG` |
| 5 | Authentication API | `passport` | `passport/.env` (copy từ `.env.local` — **vừa tạo lại, trước đó bị thiếu hoàn toàn**) | `st-mysql` | `basis` | root / `Qampa4jLoxCSoXsG` |
| 6 | Fund API | `st-api` | env truyền qua `dc-web/.env_st-app-pfund` (không có `.env` riêng trong repo) | `st-mysql` | động: `{prefix}_pfund` (tương tự st-admin) | root / `Qampa4jLoxCSoXsG` |
| 7 | Trading Site | `app-vuejs` | không kết nối DB trực tiếp — gọi qua API (4,5,6) | — | — | — |
| 8 | Account Opening Site | `mobile-registration` | không kết nối DB trực tiếp — gọi qua API | — | — | — |

**Phát hiện khi tổng hợp**: `common-front-api` và `passport` **hoàn toàn chưa có file `.env`** (chỉ có `.env.local`/`.env.example`) — đây là thiếu sót từ lúc thiết lập ban đầu (đã ghi là "copy .env.local → .env" trong mục 1.3 nhưng thực tế chưa làm cho 2 repo này). Đã khắc phục ngay: `cp .env.local .env` cho cả hai. Vì cả hai chỉ dùng route mặc định (Laravel welcome page) nên trước đó không lộ ra lỗi, nhưng bất kỳ route nào cần DB/APP_KEY sẽ lỗi nếu chưa có `.env`.

---

# local-st-admin-hsec.crudist.tech
Dự án: Fund Management Dashboard (ファンド管理画面) — repo `st-admin`. Tài khoản: `admin/123456`.

## Môi trường chạy hiện tại
- Chạy trên **Docker** (không chạy trực tiếp trên host), trong network `st_net`.
- Runtime: **PHP 7.4.33** (image `st/php:7.4-fpm`, build từ `docker--php74` + `docker-php74-fpm`).
- Container ứng dụng: `st-app-pfundadmin` (tên service trong compose là `st-app-pfund-admin`) — mount source code tại `/www/st-admin`, docroot `/www/st-admin/Webroot`.
- Reverse proxy: nginx `st-web` (image `st/nginx:1.27`), vhost `local-docker/etc/local/nginx/pfund-admin.conf`, publish port `80:80` ra host.
- DB: MySQL container `st-mysql` (`st/mysql:8.0.40`), user `root` / pass `Qampa4jLoxCSoXsG`.

## Container cần khởi động trước
Theo đúng thứ tự phụ thuộc:
1. `st_net` — network dùng chung (`docker network create st_net`, hoặc `make start-network` tại `local-docker/`).
2. Stack `dc-base` — cung cấp `st-mysql`, `st-redis` (bắt buộc phải chạy trước vì `st-app-pfundadmin` kết nối tới `st-mysql`/`st-redis` qua tên container):
   ```bash
   cd local-docker/dc-base && docker compose up -d
   ```
3. Stack `dc-web` — chứa `st-app-pfundadmin` + nginx `st-web` (khởi động cùng lúc toàn bộ 6 app PHP + nginx dùng chung):
   ```bash
   cd local-docker/dc-web && docker compose up -d
   ```
4. **Quan trọng**: đảm bảo port 80 trên máy host (WSL) không bị chiếm bởi service khác. Trên môi trường này phát hiện `apache2` (service hệ thống có sẵn trong WSL, chỉ serve trang default, không phục vụ site nào) đang chiếm port 80 khiến `st-web` không bind được cổng dù container vẫn "Up". Đã xử lý bằng:
   ```bash
   sudo systemctl stop apache2 && sudo systemctl disable apache2
   ```
   Sau khi tắt apache2, cần restart lại `st-web` để nó bind port 80: `docker compose restart st-web` (trong thư mục `dc-web`).

## Composer install (chạy 1 lần, nếu thư mục `st-admin/vendor` chưa có)
```bash
docker exec -it st-app-pfundadmin sh -c '
  mkdir -p /var/www/.composer
  cat > /var/www/.composer/auth.json <<EOF
  {"http-basic": {"gitlab.hashdash-group.com": {"username": "$GITLAB_USER", "password": "$GITLAB_PASS"}}}
EOF
  cd /www/st-admin && composer install -o --no-interaction
'
```
(`$GITLAB_USER`/`$GITLAB_PASS` đã có sẵn trong container qua `env_file: ../.env`.)

## Truy cập
- Thêm dòng sau vào file hosts (trên WSL: `/etc/hosts`; nếu trình duyệt chạy trên Windows thì cần thêm cả vào file hosts của Windows như bạn đã khai báo):
  ```
  127.0.0.1 local-st-admin-hsec.crudist.tech
  ```
- Mở: http://local-st-admin-hsec.crudist.tech/ → tự động redirect tới `?c=index&a=login`, hiển thị đúng trang đăng nhập "匿名組合ファンド管理画面" (đã xác nhận bằng curl, `Server: nginx/1.31.5`, không phải trang mặc định của apache).

## Kết quả kiểm tra
- ✅ Container khởi động ổn định, nginx trả đúng trang login của ứng dụng (không phải trang lỗi/trang mặc định).
- ✅ **[CẬP NHẬT 2026-09-04] Đăng nhập THÀNH CÔNG** với `admin/123456` sau khi import dữ liệu STG thật vào `basis.BCO_COMPANY` (xem mục 4, phần "Import dữ liệu STG thật") — lỗi `Unknown database '_pfund'` đã biến mất hoàn toàn. Vào đúng dashboard với đầy đủ menu nghiệp vụ.

---

# local-hhd-admin-hsec.crudist.tech
Dự án: Company-specific Management Dashboard (会社別管理画面) — repo `company-admin`. Tài khoản: `arakawa/1qazxsw2`.

## Môi trường chạy hiện tại
- Chạy trên **Docker**, network `st_net`.
- Runtime: **PHP 7.4.33** (image `st/php:7.4-fpm`).
- Framework: **FuelPHP 1.8.2**, vendor cài qua `fuel/vendor` (không phải `vendor/` chuẩn — do `composer.local.json` cấu hình `vendor-dir: fuel/vendor`).
- Container ứng dụng: `st-app-cmpadmin` — mount `/www/company-admin`, docroot `/www/company-admin/public`.
- DB creds không dùng file `.env` mà inject trực tiếp qua `fastcgi_param` trong vhost `company-admin.conf` (`DB_MASTER_HOST/NAME/USER/PASS`, `BASIS_DB_NAME`, `COMPANY_SEQ_NO=1`).

## Container cần khởi động trước
Giống st-admin: `dc-base` (MySQL/Redis) → `dc-web` (chứa `st-app-cmpadmin` + `st-web`) → đảm bảo port 80 không bị apache2/service khác chiếm.

## Composer install (đã cài, dùng script riêng của repo)
```bash
docker exec st-app-cmpadmin sh -c '
  mkdir -p /var/www/.composer
  cat > /var/www/.composer/auth.json <<EOF
  {"http-basic": {"gitlab.hashdash-group.com": {"username": "$GITLAB_USER", "password": "$GITLAB_PASS"}}}
EOF
  cd /www/company-admin && COMPOSER=composer.local.json ./composer-with-env-docker.sh install -o --no-interaction
'
```

## Truy cập
```
127.0.0.1 local-hhd-admin-hsec.crudist.tech
```
http://local-hhd-admin-hsec.crudist.tech/ → redirect `/login`, hiển thị đúng trang "管理画面" (xác nhận `Server: nginx/1.31.5`, không phải Apache default).

## Kết quả kiểm tra
- ✅ App chạy đúng, trang login thật hiển thị chuẩn.
- ⚠️ Đăng nhập `arakawa/1qazxsw2` ban đầu bị **fatal error** (`Array to string conversion` trong logger `fluentd`) — truy vết log thật thì lỗi gốc là `SQLSTATE[42S22]: Unknown column 'USER_TYPE'` khi query `hhd.ECO_ADMIN_USER`, do schema `hhd` migrate dở dang (lỗi `RANK`, xem mục 4). **Đã sửa lỗi migration và migrate lại xong `hhd`** — cột `USER_TYPE` đã tồn tại, query không còn lỗi.
- ⚠️ **[CẬP NHẬT 2026-09-04]** Đã import dữ liệu STG thật (xem mục 4) — `hhd.ECO_ADMIN_USER` giờ có 78 dòng, nhưng **user `arakawa` không có trong danh sách này** (danh sách thật: `admin, zhaot, yamada, saito, testadmin01, uatuser...` — có thể `arakawa` chỉ tồn tại ở production). Chưa xác nhận được đăng nhập `arakawa/1qazxsw2` do thiếu tài khoản, không phải do lỗi môi trường. Có thể thử với 1 trong các `USER_ID` thật ở trên nếu biết mật khẩu tương ứng.

---

# local-basis-admin-hsec.crudist.tech
Dự án: Common Management Dashboard / BasisAdmin (共通管理画面) — repo `basis-admin`. Tài khoản: `admin/admin1234`.

## Môi trường chạy hiện tại
- Chạy trên **Docker**, network `st_net`.
- Runtime: **PHP 7.4.33** (image `st/php:7.4-fpm`).
- Framework: **Laravel** (kiểu laravel-admin/AdminLTE).
- Container ứng dụng: `st-app-basadmin` — mount `/www/basis-admin`, docroot `/www/basis-admin/public`.
- DB: dùng file `.env` riêng của repo (copy từ `.env.local`), `DB_DATABASE=basis`.

## Container cần khởi động trước
Giống st-admin: `dc-base` → `dc-web` → đảm bảo port 80 free.

## Truy cập
```
127.0.0.1 local-basis-admin-hsec.crudist.tech
```
http://local-basis-admin-hsec.crudist.tech/ → redirect `/auth/login`, trang "Basis-admin | ログイン" (AdminLTE).

## Kết quả kiểm tra
- ✅ **Đăng nhập THÀNH CÔNG** với `admin/admin1234` — xác nhận qua CSRF token + session cookie thật, `POST /auth/login` trả `302 → /home`, và fetch `/home` với cookie trả về đúng layout dashboard (Sidebar, navbar-custom-menu của AdminLTE), không bị bounce lại trang login.
- Đây là dự án **duy nhất trong số 8 dự án hiện có thể đăng nhập và dùng được đầy đủ** ngay bây giờ, vì `basis-admin` chỉ phụ thuộc schema `basis` (đã có sẵn admin user thật từ trước, không cần dữ liệu company/hhd).

---

# local-common-front-api-hsec.crudist.tech
Dự án: Common API (共通api) — repo `common-front-api`. Không có tài khoản (API thuần, không có màn hình đăng nhập).

## Môi trường chạy hiện tại
- Chạy trên **Docker**, network `st_net`.
- Runtime: **PHP 8.3.16** (image `st/php:8.3-fpm`, xem mục 1.2 để biết đính chính version so với yêu cầu ban đầu ghi "8.4").
- Framework: **Laravel**.
- Container ứng dụng: `st-app` (dùng CHUNG với `passport`) — mount `/www/common-front-api`, docroot `/www/common-front-api/public`.
- **Phát hiện & đã sửa**: repo này ban đầu **hoàn toàn thiếu file `.env`** (chỉ có `.env.local`) — đã `cp .env.local .env` (xem mục 5).

## Container cần khởi động trước
`dc-base` → `dc-web` → đảm bảo port 80 free.

## Truy cập
```
127.0.0.1 local-common-front-api-hsec.crudist.tech
```

## Kết quả kiểm tra
- ✅ App Laravel thật đang chạy (trang welcome mặc định của Laravel vì route `/` không định nghĩa riêng — đây là API nên cần gọi đúng endpoint `/api/...` để test nghiệp vụ thật, không có UI đăng nhập).

---

# local-passport-api-hsec.crudist.tech
Dự án: Authentication API (認証api) — repo `passport`. Không có tài khoản riêng cho dự án này trong yêu cầu ban đầu.

## Môi trường chạy hiện tại
- Chạy trên **Docker**, network `st_net`.
- Runtime: **PHP 8.3.16** (image `st/php:8.3-fpm`, đính chính version tương tự Common API).
- Framework: **Laravel**.
- Container ứng dụng: `st-app` (dùng CHUNG với `common-front-api`).
- **Phát hiện & đã sửa**: cũng thiếu file `.env` hoàn toàn, đã `cp .env.local .env`.

## Container cần khởi động trước
`dc-base` → `dc-web` → đảm bảo port 80 free.

## Kết quả kiểm tra
- ✅ App Laravel thật đang chạy (session cookie thật, `X-Powered-By: PHP/8.3.16`).
- Endpoint `POST /user/login` (dùng cho app-vuejs) đã thử — yêu cầu `USERNAME`, `PASSWORD` (đã mã hoá bằng `SECRET_VERSION_ID`), và cần company đã được resolve qua header `X-Ci` → phụ thuộc `basis.BCO_COMPANY` (cùng gap dữ liệu như các dự án khác).

---

# local-st-front-api-hsec.crudist.tech
Dự án: Fund API (ファンドapi) — repo `st-api`. Không có tài khoản riêng.

## Môi trường chạy hiện tại
- Chạy trên **Docker**, network `st_net`.
- Runtime: **PHP 8.3.16** (image `st/php:8.3-fpm`).
- Container ứng dụng: `st-app-pfund` — mount `/www/st-api`, docroot `/www/st-api/Webroot`.
- `composer.lock` khóa PHP đúng `8.3.12` trong khi image là `8.3.16` — đã cài bằng `composer install --ignore-platform-reqs` (xem mục 4).

## Kết quả kiểm tra
- ✅ App thật đang chạy, trả JSON lỗi có cấu trúc (`{"STATUS":"NG",...}`) — không phải trang lỗi PHP thô.
- Test với header `X-Ci: sgDj8sHr` → truy vết log cho thấy query `SELECT SEQ_NO,DB_NAME FROM basis.BCO_COMPANY WHERE COMPANY_ID='sgDj8sHr'` trả rỗng — cùng gap dữ liệu `BCO_COMPANY` như st-admin.

---

# local-st-front-hsec.crudist.tech:8080
Dự án: Trading Site (取引サイト) — repo `app-vuejs`. Tài khoản: `10010000212/a_testing18`.

## Môi trường chạy hiện tại
- Chạy trên **Docker**, network `st_net`, **không dùng PHP** — Node 18 + webpack-dev-server.
- Stack riêng `dc-app-vuejs-dev`: container `front-app-dev` (chạy `npm run dev.local`) + container proxy `app-vuejs-nginx` (openresty, publish `8080:80`, có HTTP basic-auth).
- `.env` đã copy từ `.env.local` (trỏ đúng API local: `VUE_APP_FUND_API`, `VUE_APP_COMMON_API`, `VUE_APP_PASSPORT_API`, `VUE_APP_REGISTER_URL`).

## Container cần khởi động trước
`dc-base` → `dc-web` (cần cho các API backend) → `dc-app-vuejs-dev`:
```bash
cd local-docker/dc-app-vuejs-dev && cp .env.local .env && make npm_install && docker compose up -d
```

## Sự cố đã gặp & đã sửa
- Webpack-dev-server bị **treo ở 69% building modules** vĩnh viễn (do tranh chấp CPU lúc chạy đồng thời nhiều composer install/migration) — đã `docker restart front-app-dev`, build lại thành công (bundle `app.js` ~24MB).

## Kết quả kiểm tra
- ✅ Bundle build xong, `index.html` + `app.js` serve đúng khi gọi trực tiếp vào container (bypass proxy).
- ⚠️ Truy cập qua trình duyệt tại `:8080` vẫn bị **401 Unauthorized** — proxy openresty yêu cầu HTTP basic-auth, file `htpasswd` chỉ có hash, không có plaintext trong repo. **Cần bạn cung cấp thông tin basic-auth hoặc đồng ý để tôi tạo mới.**
- Login thật (`10010000212/a_testing18`) đã thử trực tiếp qua API `passport` — bị chặn bởi cùng gap dữ liệu `BCO_COMPANY` (chưa test được qua UI vì basic-auth).

---

# local-register-hsec.crudist.tech:8081
Dự án: Account Opening Site (口座開設サイト) — repo `mobile-registration`. Không có tài khoản (site đăng ký công khai).

## Môi trường chạy hiện tại
- Chạy trên **Docker**, Node 18 + webpack-dev-server, tương tự app-vuejs.
- Stack riêng `dc-reg-vuejs-dev`: `front-reg-dev` (chạy `npm run local`) + proxy `front-reg-dev-proxy` (openresty, publish `8081:8080`, có basic-auth).
- Không có `.env.local` — cấu hình build-time nằm ở `config/local.env.js` (đã đúng sẵn, trỏ `PASSPORT_API` local).

## Container cần khởi động trước
```bash
cd local-docker/dc-reg-vuejs-dev && cp .env.local .env && make npm_install && docker compose up -d
```

## Sự cố đã gặp & đã sửa
- Cùng bug treo ở 69% như app-vuejs — đã `docker restart front-reg-dev`, build lại thành công (bundle `app.js` ~18MB).

## Kết quả kiểm tra
- ✅ Bundle build xong, serve đúng khi bypass proxy.
- ⚠️ Cũng bị chặn bởi basic-auth (401) ở proxy `:8081` — cùng vấn đề với app-vuejs, cần thông tin basic-auth.
