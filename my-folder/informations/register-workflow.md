# Register Work flow

# Step 1: Sign up via email (メール登録 )
The user visits the homepage. Then, they click "Open Account" to proceed to the email entry page (setEmail). In the `setEmail` page, implement the following:
- Enter your email address. Then, check your inbox and click the link included in the email to go to the user information entry page (`applyTypeChoose`).

## Thông tin về các component liên quan.
1. setEmail
 - Location: [setEmail.vue](/mobile-registration/src/views/CommonResiger/setEmail.vue)
 - Chức năng chính: dùng để thực hiện gửi email xác nhận người dùng không phải là robot
 - Các API sử dụng:
   - API `send invite email`:
     - Location:[UserRequestController.php](/common-front-api/app/Http/Controllers/UserRequestController.php#L418-L426)
     - Method: POST
     - Endpoint: `/register/send_invite_email`
     - Base URL local: `http://local-common-front-api-hsec.crudist.tech/`
     - Full URL local: `http://local-common-front-api-hsec.crudist.tech/register/send_invite_email`
     - Được sử dụng ở hàm [sendCode](/mobile-registration/src/views/CommonResiger/setEmail.vue#L164-L225)
     - Backend khai báo tại: [api.php](/common-front-api/routes/api.php#L78-L90).
     - Request body [SendRegisterInviterRequest.php](/common-front-api/app/Http/Requests/SendRegisterInviterRequest.php):
        ```json
        {
          "EMAIL": "user@example.com", // registration email
          "TEL_NO": "", // số điện thoại, dùng khi đăng ký qua SMS.
          "HASH_CODE": "", // mã giới thiệu nếu người dùng đi qua link mời.
          "UTM_SOURCE": "", // Các trường UTM_*: thông tin tracking marketing, chỉ được thêm khi URL có tham số hợp lệ.
          "UTM_MEDIUM": "",
          "UTM_CAMPAIGN": "",
          "UTM_TERM": "",
          "UTM_CONTENT": "",
          "UTM_SOURCE_PLATFORM": "",
          "UTM_CREATIVE_FORMAT": "",
          "UTM_MARKETING_TACTIC": ""
        }
       ```
     - Validation: 
       - Location: [SendRegisterInviterRequest.php](/common-front-api/app/Http/Requests/SendRegisterInviterRequest.php)
       - `EMAIL` phải đúng format email.
       - `TEL_NO` phải có dạng 0 + 10 chữ số.
       - Nếu có một trong `UTM_SOURCE`, `UTM_MEDIUM`, `UTM_CAMPAIGN` thì cả ba trường bắt buộc phải có.
     - Header đặc biệt:
         - X-CI
         - X-DU
         - X-SV
         - X-PF
         - X-UA
         - X-MD
         - X-OV
         - Riêng API này thêm X-CT lấy từ cookie ct, xem httpRequest.js:118-139. Backend kiểm tra X-CT trong Redis trước khi xử lý tại [ApiController.php:37-50](/common-front-api/app/Http/Controllers/ApiController.php#L37-L50).
 - Logic front-end:
   - Người dùng nhập email.
   - checkEmail() kiểm tra format bằng thư viện email-validator.
   - Nếu email sai hoặc URL tracking không hợp lệ thì không gửi request.
   - Hiển thị loading.
   - Gọi POST /register/send_invite_email.
   - Nếu response có: `"STATUS": "OK"` thì lưu email vào local storage và chuyển đến [emailTips.vue](/mobile-registration/src/views/CommonResiger/emailTips.vue).
   - Nếu lỗi, lấy `resp.data.ERROR.MESSAGE` để hiển thị cho người dùng. Logic này nằm tại [setEmail.vue:164-225](/mobile-registration/src/views/CommonResiger/setEmail.vue#L164-L225).
 - Logic backend:
   - Controller gọi `InviterService::sendRegisterInviteMail()` tại [UserRequestController.php:418-426](/common-front-api/app/Http/Controllers/UserRequestController.php#L418-L426).
   - Service thực hiện:
     - Kiểm tra phải có EMAIL hoặc TEL_NO.
     - Kiểm tra email đã tồn tại trong hệ thống chưa.
     - Giới hạn số lần gửi lại link
     - Hủy các request cũ còn hiệu lực của cùng email/số điện thoại.
     - Tạo mã đăng ký dạng: `REQ + md5(...)`.
     - Đặt thời hạn link là 10 phút.
     - Nếu có email:
       - Gọi [sendHashMail()](/common-front-api/app/Services/InviterService.php#L108-L152).
       - Render [template emails](/common-front-api/resources/views/emails/registerUrl.blade.php).
       - Gửi email đến địa chỉ người dùng.
     - Nếu chỉ có số điện thoại-> Gửi SMS chứa link đăng ký
     - Lưu thông tin request và các tham số UTM vào database.
     - Trả về kết quả thành công.
     - Chi tiết nằm tại [InviterService.php:203-326](/common-front-api/app/Services/InviterService.php#L203-L326)
 - Logic khi người dùng click link có dạng `{WEB_REGISTER_URL}/transferPage?HASH_CODE={HASH_CODE}`:
   - Trang [transferPage](/mobile-registration/src/views/CommonResiger/transferPage.vue#L15-L78) gọi GET [/inviter/check_hash_code](/common-front-api/app/Http/Controllers/InviterController.php#L46-L92).
   - Backend kiểm tra mã còn hiệu lực không.
   - Nếu hợp lệ, lưu `HASH_CODE` vào `sessionStorage` và local storage.
   - Chuyển người dùng đến `applyTypeChoose`.
    
# Step 2: Enter personnal infomation (お客様情報の入力)
Bước 2 bao là bước nhập thông tin người dùng bao gồm nhiều trang và có 2 điều hướng chính, trang hiển thị đầu tiên là `applyTypeChoose` chức năng chính là xác định tài khoản người dùng muốn đăng ký là Cá nhân hay Doanh nghiệp để thực hiện điều hướng luồng nhập thông tin phù hợp. Với quy trình chi tiết như sau:
## 2.1 applyTypeChoose.vue. 
   - [Location](/mobile-registration/src/views/CommonResiger/applyTypeChoose.vue). 
   - Chức năng chính: Xác định loại tài khoản người dùng muốn đăng ký và thực hiện điều hướng nhập thông tin theo loại tài khoản người dùng chọn.
   - Logic front-end:
     1. Khởi tạo dữ liệu:
        - Component lấy:
          - Tiêu đề và tiến độ từ `commonJs.getTopTitle(this.$route.path)`.
          - Selection list: `commonData.applyTypeData`.
          - Loại đăng ký hiện tại từ `localStorage`.
        - Selection list:
          ```json
            [
              { id: 1, value: '個人のお申込み' }, // Invidual
              { id: 2, value: '法人のお申込み' }  // Enterprise
            ]
          ```
        - Khi vào trang, `beforeRouteEnter()` gọi `getlocalData()` để khôi phục lựa chọn trước đó `this.applyTypeId = this.commonJs.getLocalData('applyTypeId');`
        - Sau đó đánh dấu lại option đã chọn bằng [putActiveData()](/mobile-registration/src/views/CommonResiger/applyTypeChoose.vue#L78-L84).
     2. Hiển thị lựa chọn
        - Component sử dụng:
          ```vue
          <cSingleChoice
          :selectdata="applyTypeData"
          @selectClick="handelSelect"
          />
          ```
        - Khi người dùng chọn một option, hàm handelSelect() được gọi:
          ```js
          handelSelect(item) {
              this.applyTypeData = this.commonJs.clearSelected(this.applyTypeData);
              item.isActive = true;
              this.applyTypeId = item.id;
          }
          ```
        - Logic thực hiện:
          - Bỏ trạng thái active của tất cả lựa chọn.
          - Đánh dấu lựa chọn hiện tại.
          - Lưu id vào `applyTypeId`.
     3. Hiển thị cảnh báo cho doanh nghiệp
        - Nếu người dùng chọn doanh nghiệp (applyTypeId == 2), phần cảnh báo được hiển thị:
            ```vue
              <ul class="main-tips" v-if="applyTypeId == 2">
            ```
        - Nội dung cảnh báo liên quan đến:
          - Nghĩa vụ thuế tại Mỹ.
          - Quốc gia cư trú tính thuế không phải Nhật Bản.
          - Người kiểm soát thực tế có nghĩa vụ thuế ở nước ngoài.
          - Người kiểm soát thực tế thuộc nhóm Foreign PEPs.
     4. Nút Next
        - Nút Next bị disable nếu chưa chọn loại đăng ký:
            ```vue
            <cNextButton
                :disabled="!applyTypeId"
                @clickNext="goNext"
                @clickBack="goBack"
            />
            ```
        - Khi bấm Next, hàm [goNext()](/mobile-registration/src/views/CommonResiger/applyTypeChoose.vue#L92-L102) chạy với luồng điều hướng: 
          - Cá nhân (value=1): [nameBirthdaySex](/mobile-registration/src/views/ResetResiger/Individual/nameBirthdaySex.vue)
          - Doanh nghiệp (value=2): [foreignCountryCompany](/mobile-registration/src/views/Resiger/Comfrim/foreignCountryCompany.vue)
        - Giá trị lựa chọn được lưu vào local storage với key `applyTypeId`.
     5. Nút Back
        - Khi bấm [Back](/mobile-registration/src/views/CommonResiger/applyTypeChoose.vue#L90-L92) Frontend quay lại route authenticationCode.
   - Các API sử dụng:
      - [getPdfList()](/mobile-registration/src/views/CommonResiger/applyTypeChoose.vue#L62-L77) gọi API [GET /register/electronic_document](/common-front-api/app/Http/Controllers/ElectronicController.php#L18-L22).

     ** Note: Tuy nhiên, trong component hiện tại hàm này không được gọi trong created, mounted hoặc các hàm xử lý khác. Vì vậy, theo code hiện tại, khi người dùng chọn loại tài khoản thì component không gọi API, chỉ lưu dữ liệu local và điều hướng sang trang tiếp theo.
   - ***Tóm tắt luồng***:
   ```
    - transferPage
        -> kiểm tra HASH_CODE
        -> chuyển đến applyTypeChoose
    - applyTypeChoose
        -> chọn Cá nhân hoặc Doanh nghiệp
        -> lưu applyTypeId vào localStorage
        -> Cá nhân: /nameBirthdaySex
        -> Doanh nghiệp: /foreignCountryCompany
   ```
## 2.2 nameBirthdaySex.vue
   - [Location](/mobile-registration/src/views/PersonalResiger/BasicInfo/nameBirthdaySex.vue)
   - Main function: là màn hình nhập thông tin cơ bản của cá nhân trong quy trình mở tài khoản bao gồm: Họ và tên bằng Kanji, Họ và tên bằng Katakana, Ngày sinh, Giới tính.
   - Logic front-end:
     -  Khởi tạo và khôi phục dữ liệu: Gọi hàm [beforeRouteEnter](/mobile-registration/src/views/PersonalResiger/BasicInfo/nameBirthdaySex.vue#L220-L224) lấy dữ liệu được lưu dưới key nameInfo trong localStorage.stoData để:
        - Reset trạng thái lựa chọn giới tính.
        - Lưu route trước đó vào fromRouter.
        - Lấy tiêu đề và tiến độ màn hình.
        - Đọc dữ liệu cũ từ local storage.
        - Khôi phục lại: year, month, day, sex, Họ tên Kanji, Họ tên Katakana.
     - Nhập và chuẩn hóa tên
       - Tên Kanji:
         - Xóa emoji và khoảng trắng không hợp lệ.
         - Chuẩn hóa khoảng trắng.
         - Chuyển half-width sang full-width.
         - Tự động chuyển Hiragana sang Katakana.
         - Tự động điền tên Katakana tương ứng nếu tên Kanji có thể chuyển đổi.
         - Kiểm tra ký tự hợp lệ bằng [verifyInputText()](/mobile-registration/src/utils/common-js.js#L202-L278).
       - Tên Katakana:
         - Chuyển Hiragana sang Katakana.
         - Chuyển half-width sang full-width.
         - Kiểm tra bằng [fullKatakanaReg](/mobile-registration/src/utils/common-data.js#L9)
     - Ngày sinh:
       - Kiểm tra định dạng ngày bằng [this.commonData.birthreg.test(val)](/mobile-registration/src/views/PersonalResiger/BasicInfo/nameBirthdaySex.vue#L206)
       - Tính tuổi: [this.commonJs.jsGetAge(this.year, this.month, this.day)](/mobile-registration/src/views/PersonalResiger/BasicInfo/nameBirthdaySex.vue#L209)
       - So sánh tuổi với độ tuổi tối thiểu lấy từ cấu hình: [this.checkAge](/mobile-registration/src/views/PersonalResiger/BasicInfo/nameBirthdaySex.vue#L210). Nếu tuổi nhỏ hơn tuổi tối thiểu, birthformatError được đặt thành true và hiển thị checkAgeMsg. Ngày sinh được chọn thông qua Framework7 picker trong initDateTimePicker().
     - Điều kiện bật nút Next:
        - Computed property [isOver](/mobile-registration/src/views/PersonalResiger/BasicInfo/nameBirthdaySex.vue#L76-L89) kiểm tra: nameStatus1, nameStatus2, brithStatus, sex. Nút Next chỉ bật khi tất cả dữ liệu đầy đủ và hợp lệ.
     - Handle next:
       - Case 1: Đang chỉnh sửa từ `/confirmInfo`
         - Xóa trạng thái editAge.
         - Xóa parentInfo.
         - Quay lại `/confirmInfo`.
       - Case 2: Đang chỉnh sửa từ `/reset/confirmInfo`
         - Lưu trạng thái vào session storage.
         - Xóa thông tin chỉnh sửa liên quan.
         - Quay lại `/reset/confirmInfo.`
       - Case 3: Đang ở trạng thái chỉnh sửa tuổi
         - Nếu editAge tồn tại: Xóa editAge -> Xóa parentInfo -> Quay lại /confirmInfo.
       - Case 4: Đăng ký mới: chuyển sang `/nationalty`
     - Lưu dữ liệu:
       - Với trường hợp thông thường thì lưu dữ liệu vào local storage:
          ```json
          {
            sex,
            year,
            month,
            day,
            seiName,
            KanaName
          }
          ```
       -  Với trường hợp chỉnh sửa từ `/reset/confirmInfo`, dữ liệu được lưu vào session storage.
   - Các API sử dụng:
     - Kiểm tra tên tiếng Nhật:
       - [Location](/common-front-api/app/Http/Controllers/CommonController.php#L30-L39)
       - Method: `POST`
       - Endpoint: `/common/check_jp_text`
       - Request body: 
          ```json
            {
              "TEXT": "山田"
            }
          ```
   - ***Tóm tắt luồng***: 
     ```
      Mở /nameBirthdaySex
       ↓
      Khôi phục nameInfo nếu có
       ↓
      Nhập họ tên, ngày sinh, giới tính
       ↓
      Kiểm tra format ở frontend
       ↓
      POST /common/check_jp_text khi blur tên Kanji
       ↓
      Kiểm tra đủ tuổi và đủ dữ liệu
       ↓
      Nhấn Next
       ↓
      Lưu nameInfo vào storage
       ↓
      Đăng ký mới: chuyển `/nationalty`
      Chỉnh sửa: quay lại `/confirmInfo` hoặc `/reset/confirmInfo`
      ```
## 2.3 nationalty.vue
   - [Location](/mobile-registration/src/views/ResetResiger/Individual/nationalty.vue)
   - Main function: là màn hình chọn quốc tịch trong luồng reset/re-apply hồ sơ cá nhân. Người dùng có thể chọn:
      ```json
        [
          { id: 1, value: '日本' },       // Nhật Bản
          { id: 2, value: '米国人等' },   // Mỹ hoặc tương đương
          { id: 9, value: 'それ以外' }    // Quốc gia khác
        ]
      ```
      - Danh sách được định nghĩa trong [common-data.js](/mobile-registration/src/utils/common-data.js#L30-L34).
      - Nếu chọn quốc gia khác (id == 9), người dùng phải nhập tên quốc gia.
   - Logic front-end:
     - Khởi tạo và khôi phục dữ liệu:
       - Trong [beforeRouteEnter()](/mobile-registration/src/views/ResetResiger/Individual/nationalty.vue#L71-L75) getlocalData() thực hiện:
         - Lưu route trước đó vào comeRouter.
         - Lấy tiêu đề màn hình.
         - Đọc `NATIONALTY` từ dữ liệu reset.
         - Đánh dấu lại quốc tịch đã chọn.
         - Nếu quốc tịch là 9, khôi phục thêm `COUNTRY_NAME`. Dữ liệu được đọc bằng:
          ```js
            this.commonJs.getResetFieldVal('NATIONALTY');
            this.commonJs.getResetFieldVal('COUNTRY_NAME');
          ```
     - Chọn quốc tịch: Khi người dùng chọn option [handelSelect()](/mobile-registration/src/views/ResetResiger/Individual/nationalty.vue#L99-L103) sẽ:
       - Bỏ trạng thái chọn cũ.
       - Đánh dấu option mới.
       - Cập nhật `nationaltyId`. 
     - Hiển thị cảnh báo quốc tịch Mỹ: Khi nationaltyId == 2, frontend hiển thị cảnh báo rằng không thể mở tài khoản do yêu cầu xử lý thuế. Tuy nhiên, option này không thể đi tiếp vì isOver trả về false. 
     - Nhập tên quốc gia khác: 
       - Khi chọn nationaltyId == 9, trường nhập quốc gia được hiển thị.
       - Khi blur, [handleBlur()](/mobile-registration/src/views/ResetResiger/Individual/nationalty.vue#L113-L120) kiểm tra tên quốc gia. Nếu không hợp lệ trả về lỗi [NATIONALTY_JP](/mobile-registration/src/utils/custom_language_jp.js#L43).
     - Điều kiện bật Next [isOver()](/mobile-registration/src/views/ResetResiger/Individual/nationalty.vue#L59-L69):
       - Nút Next chỉ được bật khi chọn Nhật Bản (id == 1), hoặc chọn quốc gia khác (id == 9) và nhập tên hợp lệ.
       - Chọn Mỹ (id == 2) sẽ không thể bấm Next.
     - Logic khi bấm Next hàm [handleNext()](/mobile-registration/src/views/ResetResiger/Individual/nationalty.vue#L121-L128) Xử lý gồm:
       - Lưu mã quốc tịch vào dữ liệu reset.
       - Xóa `COUNTRY_NAME` cũ để tránh giữ dữ liệu lỗi thời.
       - Nếu chọn quốc gia khác, lưu lại tên quốc gia.
       - Chuyển tới /resetConfirmInfo.
     - Dữ liệu được lưu bằng `setResetFieldVal()`, trong nhóm `resetData` của `localStorage`.
     - Logic nút Back:
       - Nếu vào từ `/confirmInfo`: Hiển thị dialog xác nhận trước khi quay lại /confirmInfo.
       - Nếu là luồng bình thường: Quay về trang trước đó.
   - Tóm tắt luồng:
     ```json
      /resetConfirmInfo
        ↓
      Người dùng chọn sửa Quốc tịch
        ↓
      /resetNationalty
        ↓
      Khôi phục NATIONALTY và COUNTRY_NAME
        ↓
      Chọn Nhật Bản hoặc quốc gia khác
        ↓
      Nếu quốc gia khác: nhập và validate COUNTRY_NAME
        ↓
      Lưu NATIONALTY và COUNTRY_NAME vào resetData
        ↓
      /resetConfirmInfo
     ```
## 2.4 address.vue
   - [Location](/mobile-registration/src/views/ResetResiger/Individual/address.vue)
   - Main function: là màn hình nhập và chỉnh sửa địa chỉ cá nhân trong luồng reset/re-apply hồ sơ. Màn hình xử lý:
     - Mã bưu điện.(Component sử dụng component con [cAddress.vue](/mobile-registration/src/components/Container/cAddress.vue) để nhập và tra cứu địa chỉ từ mã bưu điện.)
     - Tỉnh/thành phố (都道府県).
     - Thành phố/quận/huyện.
     - Số nhà.
     - Tên tòa nhà/phòng.
     - Địa chỉ tại thời điểm ngày 1/1 của năm hiện tại nếu khác địa chỉ hiện tại.
   - Logic frontend:
     - Khôi phục dữ liệu:
       - Trong [beforeRouteEnter()](/mobile-registration/src/views/ResetResiger/Individual/address.vue#L157-L161) hàm `getInitialData()` đọc dữ liệu reset hiện có sau đó khôi phục:\
         - POSTAL_CD
         - ADDRESS1
         - ADDRESS2
         - ADDRESS3
         - ADDRESS4
         - ADDRESS4_KANA
         - AREA_CD
         - STATE_CD
         - STATE_ADDRESS2
       - Component cũng xác định địa chỉ hiện tại có trùng với địa chỉ tại ngày 1/1 hay không:
          ```js 
          this.selected =
              parseInt(data.STATE_CD) == parseInt(data.AREA_CD)
                ? this.nowAddressData[0]
                : this.nowAddressData[1];
          ```
          - Hai lựa chọn: 1: Địa chỉ ngày 1/1 giống địa chỉ hiện tại, 2: Địa chỉ ngày 1/1 khác địa chỉ hiện tại.
          - Nếu chọn 2, người dùng phải nhập thêm: Tỉnh/thành phố tại thời điểm đó, thành phố/quận/huyện tại thời điểm đó.
     - Nhập và tra cứu địa chỉ:
       - cAddress nhận object địa chỉ từ component cha.
       - Khi dữ liệu trong cAddress thay đổi, updateAddress() cập nhật.
       - Component cAddress hỗ trợ:
           - Nhập mã bưu điện 7 chữ số.
           - Gọi API tra cứu địa chỉ.
           - Hiển thị danh sách kết quả.
           - Người dùng chọn kết quả phù hợp.
           - Tự động điền tỉnh, thành phố và khu vực.
     - Điều kiện bật nút Next:
       - Tất cả trường bắt buộc không có lỗi format.
       - Mã bưu điện phải có dữ liệu.
       - Tỉnh/thành phố phải được chọn.
       - Thành phố/quận/huyện phải có dữ liệu.
       - Người dùng phải chọn câu trả lời về địa chỉ ngày 1/1.
       - Nếu chọn địa chỉ ngày 1/1 khác địa chỉ hiện tại:
         - Phải chọn tỉnh/thành phố.
         - Phải nhập thành phố/quận/huyện.
         - Không được có lỗi format.
       - Nếu có nhập tên tòa nhà/phòng thì phải có cả tên phòng và tên phòng Katakana theo điều kiện hiện tại của code.
     - Lưu dữ liệu khi bấm Next:
        - Hàm `saveData()` lưu các trường vào `resetData`:
          - POSTAL_CD
          - ADDRESS1
          - ADDRESS2
          - ADDRESS3
          - ADDRESS4
          - ADDRESS4_KANA
          - AREA_CD
          - STATE_CD
          - STATE_ADDRESS2
          - ADDRESS3_KANA
        - Nếu địa chỉ ngày 1/1 giống địa chỉ hiện tại:
            ```js
              STATE_CD = AREA_CD;
              STATE_ADDRESS2 = '';
            ```
          Nếu khác:
            ```js
              STATE_CD = ad_nowaddress.addressId;
              STATE_ADDRESS2 = ad_nowaddress.address2;
            ```
     - Logic nút Back:
       - Nếu vào từ `/confirmInfo`: Hiển thị dialog xác nhận trước khi quay lại /confirmInfo.
       - Nếu là luồng bình thường: Quay về trang trước đó.
   - Các API sử dụng:
     - API tra cứu địa chỉ theo mã bưu điện
       - [Location](/common-front-api/app/Http/Controllers/AddressController.php)
       - Method: `GET`
       - Endpoint: `/search/address`
       - Query parameter: 
         ```json
          {
            "zipcode": "1020073"
          }
         ```
       - Response thành công được dùng để tạo danh sách địa chỉ.
       - Sau khi người dùng chọn một kết quả, component tự điền:
         - address1
         - address2
         - searchedPostal
         - area_cd
     - API kiểm tra ký tự địa chỉ
       - [Location](/common-front-api/app/Http/Controllers/CommonController.php#L30-L38)
       - Method: `POST`
       - Endpoint: `/common/check_jp_text`
       - Request body: 
         ```json
          {
            "TEXT": "千代田区九段北"
          }
         ```
       - 
   - Tóm tắt luồng:
      ```
        /reset/address
          ↓
        Đọc dữ liệu địa chỉ từ resetData
          ↓
        Nhập mã bưu điện
          ↓
        GET /search/address?zipcode=...
          ↓
        Chọn địa chỉ từ kết quả tra cứu
          ↓
        Nhập hoặc chỉnh sửa số nhà, tên tòa nhà
          ↓
        Chọn địa chỉ ngày 1/1 giống hoặc khác hiện tại
          ↓
        Nếu khác, nhập thêm địa chỉ cũ
          ↓
        Validate toàn bộ dữ liệu
          ↓
        Lưu vào resetData
          ↓
        Chuyển về trang xác nhận reset
      ```
## 2.5 tel.vue
   - [Location](/mobile-registration/src/views/ResetResiger/Individual/tel.vue)
   - Main function: là màn hình nhập và chỉnh sửa thông tin liên lạc trong luồng reset/re-apply hồ sơ cá nhân. Component xử lý hai loại số điện thoại: 
     - Số điện thoại di động: bắt buộc.
     - Số điện thoại cố định: không bắt buộc.
   - Logic frontend:
     - Khôi phục dữ liệu cũ:
       - Hàm [beforRouterEnter()](/mobile-registration/src/views/ResetResiger/Individual/tel.vue#L71-L75) thực hiện đọc dữ liệu từ resetData(), sau đó lấy số điện thoại từ cấu trúc dữ liệu có trạng thái lỗi với ý nghĩa là:
         - Nếu trường có FAILED, input được để trống.
         - Nếu hợp lệ, lấy giá trị từ DATA.
     - Kiểm tra số di động + số cố định:
       - Số di động được kiểm tra trong hàm [blurInput(0)](/mobile-registration/src/views/ResetResiger/Individual/tel.vue#L84-L115) với logic như sau:
         - Parse số điện thoại theo vùng Nhật Bản bằng hàm `parsePhoneNumberFromString(value, 'JP')` của thư viện `libphonenumber-js`
         - Lấy loại số điện thoại: `parseMax(phoneNumber.formatNational(), 'JP').getType()`.
         - Chấp nhận khi:
            ```js
              phoneNumber.isValid()
              && type == 'MOBILE'
              && this.commonJs.VerifyTel(value)
            ```
            [VerifyTel()](/mobile-registration/src/utils/common-js.js#L156-L159) kiểm tra regex `/^0\d{10}$/` => Tức là số điện thoại phải có dạng 0 + 10 chữ số, tổng cộng 11 chữ số.
     - Điều kiện bật nút Next. Nút Next chỉ bật khi:
       - Có số di động.
       - Số di động hợp lệ.
       - Nếu có nhập số cố định thì số cố định cũng phải hợp lệ.
     - Logic nút Back:
       - Nếu vào từ `/confirmInfo`: Hiển thị dialog xác nhận trước khi quay lại /confirmInfo.
       - Nếu là luồng bình thường: Quay về trang trước đó.
     - Lưu dữ liệu:
       - Khi nhấn Next hàm [handelNext()](/mobile-registration/src/views/ResetResiger/Individual/tel.vue#L137-L141) sẽ thực hiện dữ liệu được lưu vào `resetData`. Component không submit dữ liệu lên backend ở bước này.
   - Tóm tắt luồng:
     ```
      /reset/tel
        ↓
      Đọc TEL_NO_02 và TEL_NO_01 từ resetData
        ↓
      Nhập số di động bắt buộc
        ↓
      Nhập số cố định không bắt buộc
        ↓
      Validate bằng libphonenumber-js
        ↓
      Kiểm tra loại MOBILE hoặc FIXED_LINE
        ↓
      Kiểm tra regex số điện thoại Nhật Bản
        ↓
      Lưu TEL_NO_02 và TEL_NO_01 vào resetData
        ↓
      /resetConfirmInfo?rt=...
     ```
## 2.6 financialBank
   - Location: [bank.vue](/mobile-registration/src/views/PersonalResiger/Financial/bank.vue)
   - Main function: là màn hình nhập tài khoản ngân hàng nhận tiền/rút tiền trong luồng mở tài khoản cá nhân. Người dùng cần nhập: 
     - Tên ngân hàng.
     - Tên chi nhánh.
     - Loại tài khoản:
        - 普通預金 (Ordinary deposit)
        - 当座預金 (Checking account)
        - 貯蓄預金 (Savings deposits)
     - Số tài khoản 7 chữ số.
   - Logic frontend: 
     - Khôi phục dữ liệu
       - Nếu có dữ liệu, [giveData()](/mobile-registration/src/views/PersonalResiger/Financial/bank.vue#L165-L180) khôi phục: 
         - organizationName
         - branch
         - accountType
         - accountNum
         - selectedBranchData
         - selectedOrganizeData
       - Dữ liệu được lưu trong `localStorage.stoData.back_obj`
     - Chọn ngân hàng: Khi người dùng nhập tên ngân hàng, getOrganizeData() được gọi và hiển thị danh sách gợi ý trong cSearchInput. Khi chọn ngân hàng hàm [handelSelectBank(item)](/mobile-registration/src/views/PersonalResiger/Financial/bank.vue#L226-L241) sẽ thực hiện:
       - Lưu tên và thông tin ngân hàng.
       - Kiểm tra `ONLINE_ORDINARY`.
       - Reset chi nhánh, loại tài khoản và số tài khoản.
       - Nếu ONLINE_ORDINARY == 1, hệ thống chỉ cho phép loại tài khoản mặc định:
          ```js
            { id: 1, value: FINANCIAL_BANK_JP.ACCOUNT_TYPE_NAME }
          ```
       - Backend hiện đánh dấu ngân hàng có mã 9900 là ONLINE_ORDINARY = 1.
     - Chọn chi nhánh:
       - Khi nhập tên chi nhánh, `getBranchData()` được gọi. Nếu chưa chọn ngân hàng, frontend hiển thị cảnh báo:
          ```js
            this.showAlert(FINANCIAL_BANK_JP.ERR_MSG);
          ```
       - Khi chọn chi nhánh: [handleSelectBranch(item)](/mobile-registration/src/views/PersonalResiger/Financial/bank.vue#L280-L283) Component lưu:
         - Tên chi nhánh vào branch.
         - Mã chi nhánh và thông tin khác vào `selectedBranchData`.
         - Nếu người dùng đổi ngân hàng, watcher selectedOrganizeData sẽ xóa chi nhánh đã chọn trước đó.
     - Chọn loại tài khoản:
          ```js
          handelSelectType(item) {
          this.accountTypeData = this.commonJs.clearSelected(this.accountTypeData);
          item.isActive = true;
          this.accountType = item;
          }
          ```
          Component chỉ cho phép chọn loại tài khoản sau khi danh sách loại tài khoản đã được thiết lập cho ngân hàng.
     - Kiểm tra số tài khoản
        - Số tài khoản phải có đúng 7 chữ số:
            ```js
              let statusAccount = this.accountNum.length == 7;
            ```
        - Khi blur Component:
          - Chuyển đổi số half-width sang full-width nếu cần bằng ToCDB().
          - Kiểm tra độ dài có đúng 7 chữ số hay không.
     - Điều kiện bật nút Next. Nút Next chỉ bật khi:
       - Đã chọn ngân hàng.
       - Đã chọn chi nhánh.
       - Đã chọn loại tài khoản.
       - Số tài khoản có 7 chữ số.
     - Logic nút Back:
       - Nếu vào từ `/confirmInfo`: Hiển thị dialog xác nhận trước khi quay lại /confirmInfo.
       - Nếu là luồng bình thường: Quay về trang trước đó.
   - Các API sử dụng:
     - API tìm kiếm ngân hàng
       - Location: [BankController.php](/common-front-api/app/Http/Controllers/BankController.php#L12-L36)
       - Method: `GET`
       - Endpoint: `/`search/withdraw_facil_name`
       - Query:
         ```json
            {
              "withdraw_facil_name": "三菱",
              "withdraw_facil_cd": ""
            }
         ```
       - Frontend gọi tại [getOrganizeData()](/mobile-registration/src/views/PersonalResiger/Financial/bank.vue#L243-L268).
       - Logic API:
         - Tìm ngân hàng theo tên.
         - Trả về mã và tên ngân hàng.
         - Gắn thêm ONLINE_ORDINARY.
         - Ngân hàng mã 9900 được đánh dấu là chỉ hỗ trợ loại tài khoản online/普通預金.
     - API tìm kiếm chi nhánh
       - Location: [BankController.php](/common-front-api/app/Http/Controllers/BankController.php#L38-L56).
       - Method: `GET`
       - Endpoint: `/search/withdraw_sub_name`
       - Query:
          ```json
            {
              "withdraw_sub_name": "本店",
              "withdraw_facil_cd": "0005"
            }
          ```
       - Frontend gọi tại [getBranchData()](/mobile-registration/src/views/PersonalResiger/Financial/bank.vue#L284-L309).
       - Logic API:
         - Nhận mã ngân hàng.
         - Tìm chi nhánh theo tên.
         - Trả về:
           - `WITHDRAW_SUB_NAME`
           - `WITHDRAW_SUB_CODE`
         - Frontend chuyển dữ liệu về dạng dùng chung:
            ```js
              {
                value: item.WITHDRAW_SUB_NAME,
                id: item.WITHDRAW_SUB_CODE
              }
            ```
   - Lưu dữ liệu và điều hướng:
     - Khi nhấn Next: dữ liệu sẽ được lưu vào localStorage chuyển hướng đến trang `occupationType`.
   - Tóm tắt luồng:
     ```
        /financialBank
          ↓
        Nhập tên ngân hàng
          ↓
        GET /search/withdraw_facil_name
          ↓
        Chọn ngân hàng
          ↓
        Nhập tên chi nhánh
          ↓
        GET /search/withdraw_sub_name
          ↓
        Chọn chi nhánh
          ↓
        Chọn loại tài khoản
          ↓
        Nhập số tài khoản 7 chữ số
          ↓
        Lưu back_obj vào localStorage
          ↓
        Luồng mới: /occupationType
        Luồng chỉnh sửa: /confirmInfo
     ```  
## 2.7 occupationType.vue
   - Location: [occupationType.vue](/mobile-registration/src/views/PersonalResiger/Occupation/occupationType.vue).
   - Main function: là màn hình chọn nghề nghiệp của khách hàng cá nhân trong quy trình mở tài khoản. Danh sách nghề nghiệp được định nghĩa trong [common-data.js](/mobile-registration/src/utils/common-data.js#L117-L127).
   - Logic frontend:
     - Khởi tạo và khôi phục dữ liệu. Hàm [beforeRouteEnter()](/mobile-registration/src/views/PersonalResiger/Occupation/occupationType.vue#L44-L47) dùng `getInitialData()` để:
       - Lưu route trước đó vào fromRouter.
       - Lấy tiêu đề và tiến độ màn hình.
       - Đọc nghề nghiệp đã lưu:
          ```js
            let data = this.commonJs.getLocalLowerData(
              'occuptionType',
              'occuption'
            );
          ```
       - Nếu có dữ liệu cũ:
         - Gán vào selectType.
         - Đánh dấu option tương ứng là active.\
       - Dữ liệu được lưu trong cấu trúc: `localStorage.stoData.occuption.occuptionType`
     - Chọn nghề nghiệp: Component dùng `cBigChoice` để thực hiện, khi người dùng chọn thì [handelSelectType](/mobile-registration/src/views/PersonalResiger/Occupation/occupationType.vue#L136-L143) sẽ thực hiện:
       - Bỏ lựa chọn cũ.
       - Lưu option hiện tại vào `selectType`.
       - Đánh dấu option đang chọn.
     - Điều kiện bật Next: Nút Next chỉ bật khi người dùng đã chọn một nghề nghiệp.
     - Logic điều hướng khi nhấn Next: Hàm xử lý chính [handelNext](/mobile-registration/src/views/PersonalResiger/Occupation/occupationType.vue#L63-L112).
       - Trường hợp đang chỉnh sửa từ `/confirmInfo`:
         - Nếu chọn nhóm id thuộc: [7, 8, 9, 10] đây là các nhóm không cần nhập thông tin nơi làm việc. Component:
           - Lưu nghề nghiệp.
           - Tắt cờ editOccuption.
           - Xóa dữ liệu nơi làm việc cũ.
           - Quay lại `/confirmInfo`.
         - Nếu chọn các nghề nghiệp khác:
           - Đặt editOccuption = true.
           - Lưu tạm nghề nghiệp vào sessionStorage.temp_occuptiontype.
           - Chuyển đến `/employmentName`.
       - Trường hợp đang ở luồng bình thường
         - Component lưu nghề nghiệp:
            ```js
              this.commonJs.saveLocalLowerData(
                'occuptionType',
                this.selectType,
                'occuption'
              );
            ```
         - Sau đó kiểm tra cờ:
            ```js
              let editOccuption =
              this.commonJs.getLocalData('editOccuption');
            ```
         - Nếu đang trong trạng thái chỉnh sửa:
           - Với id 7, 8, 9, 10:
             - Xóa trạng thái chỉnh sửa.
             - Xóa dữ liệu nơi làm việc.
             - Quay lại /confirmInfo
           - Với các nghề nghiệp còn lại:
             - Giữ editOccuption = true.
             - Chuyển đến /employmentName.
         - Nếu là luồng đăng ký mới
           - Với id 7, 8, 9, 10:
             - Bỏ qua bước nhập nơi làm việc.
             - Chuyển đến /questionMotiveType.
           - Với các nghề nghiệp còn lại:
             - Chuyển đến `/employmentName`.  
     - Logic nút Back:
       - Nếu vào từ `/confirmInfo`: Hiển thị dialog xác nhận trước khi quay lại /confirmInfo.
       - Nếu là luồng bình thường: Quay về trang trước đó `financialBank`.
     - Tóm tắt luồng:
        ```
          /occupationType
            ↓
          Đọc nghề nghiệp đã lưu nếu có
            ↓
          Người dùng chọn nghề nghiệp
            ↓
          Kiểm tra đã chọn option hay chưa
            ↓
          Lưu vào localStorage.stoData.occuption
            ↓
          Nếu là nội trợ/sinh viên/thất nghiệp/tự doanh:
                bỏ qua thông tin nơi làm việc
                → /questionMotiveType

          Nếu là nghề cần thông tin công ty:
                → /employmentName
        ```
## 2.8 employmentName.vue
   - Location:[employmentName.vue](/mobile-registration/src/views/PersonalResiger/Occupation/employmentName.vue).
   - Main function: là màn hình nhập thông tin nơi làm việc của khách hàng cá nhân. Màn hình gồm:
     - Tên công ty/nơi làm việc.
     - Phòng ban.
     - Chức vụ. 
     - Ngoài ra có link: `金融機関にお勤めの方へのご注意` để mở trang hướng dẫn `subpagetips`.
   - Logic frontend:
     - Khôi phục dữ liệu: Trong [beforeRouteEnter()](/mobile-registration/src/views/PersonalResiger/Occupation/employmentName.vue#L121-L125) hàm `getInitialData()` thực hiện:
        - Lưu route trước đó vào fromRouter.
        - Lấy tiêu đề màn hình.
        - Đọc nghề nghiệp đã chọn bằng `getOccuption()`.
        - Đọc dữ liệu nơi làm việc từ:
          ```js
            this.commonJs.getLocalLowerData(
              'employname_obj',
              'occuption'
            );
          ```
        - Dữ liệu được khôi phục gồm:
          - employment
          - department
          - position
          - business
        - Dữ liệu thuộc nhóm:`ocalStorage.stoData.occuption.employname_obj`
     - Xác định loại nghề nghiệp: `occuptionType` được dùng để biết loại nghề nghiệp hiện tại. Phần chọn ngành nghề business hiện đã bị comment trong template, nên không hiển thị thực tế.
     - Validation frontend:
       - Ba trường sau đều được yêu cầu nhập: ['employment', 'department', 'position']
       - Computed isOver chỉ trả về true khi:
         - Cả ba trường đều có dữ liệu.
         - Không trường nào có lỗi format.
       - Kiểm tra khi blur
         - Khi rời khỏi input, handleBlue(key) gọi:
           ```js
            this.commonJs.verifyInputText(value, 5)
           ```
           Kiểu validation 5 cho phép nhóm ký tự như: 
             - iragana.
             - Katakana.
             - Kanji.
             - Chữ cái.
             - Số.
             - Một số ký tự đặc biệt hợp lệ.
           Nếu không hợp lệ:
              ```js
                this.errorMsg[key].error = true;
                this.errorMsg[key].msg = COMMON_JP.MES_TEXT_1;
              ```
       - Kiểm tra khi nhập
         - handleInput(value, key) cũng thực hiện cùng validation:
            ```js
            if (this.commonJs.verifyInputText(value, 5)) {
              this.errorMsg[key].error = false;
            } else {
              this.errorMsg[key].error = true;
            }
            ```
     - Logic khi nhấn Next: 
       - Khi nhấn Next, component tạo object:
          ```js
            let employname_obj = {
              employment: this.employment,
              department: this.department,
              position: this.position,
              business: this.occuptionType != 5
                ? this.business
                : {}
            };
          ```
          - Sau đó lưu vào: `localStorage.stoData.occuption.employname_obj` 
          - Thông qua:
            ```js
              this.commonJs.saveLocalLowerData(
                'employname_obj',
                employname_obj,
                'occuption'
              );
          ```
       - Trường hợp đang chỉnh sửa
         - Nếu editOccuption đang bật: 
           - Lấy loại nghề nghiệp tạm từ: `sessionStorage.temp_occuptiontype`.
           - Ghi lại loại nghề nghiệp vào local storage.
           - Tắt cờ editOccuption.
           - Xóa dữ liệu tạm.
           - Quay lại /confirmInfo.
              ```js
                this.commonJs.saveLocalLowerData(
                  'occuptionType',
                  JSON.parse(
                    sessionStorage.getItem('temp_occuptiontype')
                  ),
                  'occuption'
                );

                this.commonJs.saveLocalData('editOccuption', false);
                sessionStorage.removeItem('temp_occuptiontype');

                return this.$router.push('/confirmInfo');
              ```
         - Trường hợp vào từ /confirmInfo: 
           - Nếu không có editOccuption nhưng fromRouter == '/confirmInfo', component chỉ quay lại: `this.$router.push('/confirmInfo');`
         - Trường hợp đăng ký mới 
           - Nếu là luồng bình thường, component chuyển đến:
              ```js
                this.commonJs.toNext(
                  '/questionMotiveType',
                  null,
                  true
                );
              ```
     - Logic nút Back:
       - Nếu vào từ `/confirmInfo`: Hiển thị dialog xác nhận trước khi quay lại /confirmInfo.
       - Nếu là luồng bình thường: Quay về trang trước đó.
   - Tóm tắt luồng
      ```
        /occupationType
         ↓
        Chọn nghề nghiệp cần thông tin nơi làm việc
          ↓
        /employmentName
          ↓
        Nhập tên công ty
          ↓
        Nhập phòng ban
          ↓
        Nhập chức vụ
          ↓
        Validate dữ liệu frontend
          ↓
        Lưu employname_obj vào localStorage
          ↓
        /questionMotiveType
      ```
## 2.9 questionMotiveType.vue
   - Location: [questionMotiveType.vue](/mobile-registration/src/views/PersonalResiger/Invest/questionMotiveType.vue).
   - Main function: là màn hình để người dùng chọn động cơ đăng ký tài khoản. Các lựa chọn được định nghĩa trong [questionMotiveData](/mobile-registration/src/utils/common-data.js#L156-L161)
   - Logic frontend:
     - Khởi tạo và khôi phục dữ liệu: beforeRouteEnter() gọi getInitialData() để:
       - Lưu route trước đó vào fromRouter.
       - Lấy tiêu đề màn hình.
       - Đọc dữ liệu động cơ đã lưu:
         ```js
          let data = this.commonJs.getLocalLowerData(
            'questionMotive',
            'invest'
          );
         ```
       - Khôi phục:
         - motive_type
         - campaign_code
         - Trạng thái active của option đã chọn. 
         - Dữ liệu nằm trong: `localStorage.stoData.invest.questionMotive`.
     - Chọn động cơ  
        ```js
        handelSelect(item) {
          this.motiveData =
            this.commonJs.clearSelected(this.motiveData);

          this.motive_type = item;
          item.isActive = true;
        }
        ```
        - Logic:
         - Bỏ trạng thái active của lựa chọn cũ.
         - Gán option mới vào motive_type.
         - Đánh dấu option đang chọn.
     - Điều kiện bật Next: Trong giao diện hiện tại, người dùng chỉ chọn motive_type. Vì vậy Next được bật khi đã chọn một động cơ.
     - Logic khi nhấn Next: 
       - Luồng đăng ký mới
          - Tạo object gồm:
            - motive_type
            - campaign_code
          - Lưu vào: `localStorage.stoData.invest.questionMotive`
          - Chuyển tới: `/purpose`
       - Luồng chỉnh sửa:
         - Nếu người dùng đi từ /confirmInfo:
           - Lưu dữ liệu mới.
           - Quay lại /confirmInfo.
           - Không đi tiếp tới /purpose.
     - Logic nút Back:
       - Nếu vào từ `/confirmInfo`: Hiển thị dialog xác nhận trước khi quay lại /confirmInfo.
       - Nếu là luồng bình thường: Quay về trang trước đó.
   - Tóm tắt luồng:
      ```
        /occupationType
          ↓
        /employmentName (nếu cần)
          ↓
        /questionMotiveType
          ↓
        Chọn động cơ đăng ký
          ↓
        Lưu invest.questionMotive
          ↓
        /purpose
        ```
## 2.10 purpose.vue
   - Location: [purpose.vue](/mobile-registration/src/views/PersonalResiger/Invest/purpose.vue)
   - Main function: là màn hình để người dùng chọn mục đích đầu tư. Các lựa chọn được định nghĩa trong [commonData.purposeData](/mobile-registration/src/utils/common-data.js#L163-L167)
   - Logic frontend: 
     - Khởi tạo và khôi phục dữ liệu
       - Component dùng beforeRouteEnter():
          ```js
            beforeRouteEnter(to, from, next) {
              next((vm) => vm.getInitialData(from.path));
            }
          ```
       - getInitialData() thực hiện:
         - Lưu route trước đó vào fromRouter.
         - Lấy tiêu đề và phần trăm tiến độ bằng:
           ```js
            this.commonJs.getTopTitle(this.$route.path);
           ```
         - Đọc dữ liệu đã lưu:
           ```js
            let data = this.commonJs.getLocalLowerData(
              'purpose',
              'invest'
            );
           ```
         - Nếu có dữ liệu cũ:
           - Khôi phục purpose.
           - Đánh dấu option tương ứng active bằng putActiveData().
         - Dữ liệu nằm tại: `localStorage.stoData.invest.purpose`.
     - Chọn mục đích đầu tư:
       - Khi người dùng chọn option:
         ```js
          handelSelect(item) {
            this.purposeData =
              this.commonJs.clearSelected(this.purposeData);

            item.isActive = true;
            this.purpose = item;
          }
         ```
         - Logic:
           - Bỏ trạng thái active của option cũ.
           - Đánh dấu option hiện tại.
           - Lưu option vào purpose.
     - Điều kiện bật Next: Nút Next chỉ bật khi có một mục đích đầu tư được chọn
     - Logic khi nhấn Next
       - Luồng đăng ký mới
         - Lưu lựa chọn vào: `localStorage.stoData.invest.purpose`
         - Chuyển sang `/annualIncome`
       - Luồng chỉnh sửa, nếu đi vào từ `/confirmInfo`
         - Lưu lựa chọn mới.
         - Quay lại `/confirmInfo`.
         - Không đi tiếp sang `/annualIncome`.
     - Logic nút Back:
       - Nếu vào từ `/confirmInfo`: Hiển thị dialog xác nhận trước khi quay lại /confirmInfo.
       - Nếu là luồng bình thường: Quay về trang trước đó. 
   - Tóm tắt luồng:
      ```
        /questionMotiveType
          ↓
        Chọn động cơ đăng ký
          ↓
        /purpose
          ↓
        Chọn mục đích đầu tư
          ↓
        Lưu invest.purpose
          ↓
        /annualIncome
      ```
## 2.11 annualIncome.vue
   - Location: [annualIncome.vue](/mobile-registration/src/views/PersonalResiger/Invest/annualIncome.vue)
   - Main function: là component dùng chung cho nhiều màn hình thông tin tài chính và kinh nghiệm đầu tư, không chỉ xử lý riêng thu nhập năm. 
     - Các route dùng chung component này:
       - `/annualIncome`: Thu nhập hằng năm
       - `/revenue`:	Loại thu nhập chính
       - `/finaDerivative`:	Tài sản tài chính
       - `/experDerivative`:	Kinh nghiệm giao dịch cổ phiếu
       - `/experienceDerivative`:	Kinh nghiệm phái sinh
       - `/expeDerivative`:	Kinh nghiệm quỹ/tín dụng/trái phiếu/kim loại quý
     - Dữ liệu lựa chọn: 
       - Component ánh xạ route hiện tại tới danh sách dữ liệu trong [commonData](/mobile-registration/src/utils/common-data.js)
         ```js
            selectNameArr: [
            'IncomeData',
            'revenueData',
            'assetData',
            'experienceData',
            'derivativeData',
            'experienceFundData'
          ]
         ```
   - Logic frontend:
     - Xác định màn hình hiện tại
       - Trong getInitialData(): 
          ```js
            this.getRouterName();
          ``` 
         - getRouterName() lấy tên route hiện tại: 
          ```js
            let routerName = this.$route.path;
          ```
         - Sau đó loại bỏ dấu / và tìm index trong:
          ```js
            routerArr: [
              'annualIncome',
              'revenue',
              'financialAssts',
              'experienceStock',
              'experienceDerivative',
              'experienceFund'
            ]
          ```
         - Kết quả được lưu vào `routerIndex`.
     - Chọn dữ liệu hiển thị: 
       - Sau khi xác định routerIndex, component lấy danh sách tương ứng:
          ```js
            this.selectData =
            this.commonData[this.selectNameArr[this.routerIndex]];
          ```
         thì component hiển thị `commonData.IncomeData`.
     - Khôi phục dữ liệu cũ:
       - Khi component được mở:
        ```js
          beforeRouteEnter(to, from, next) {
            next((vm) => vm.getInitialData(from.path));
          }
        ```
       - Sau khi xác định routerIndex, component lấy dữ liệu đã lưu bằng key tương ứng:
          ```js
            this.localNameArr[this.routerIndex]
          ``` 
       - Danh sách key:
          ```js
           localNameArr: [
              'income',
              'revenuetype',
              'asset',
              'experience',
              'experienceDerivative',
              'experienceFund'
            ] 
          ```
           Dữ liệu được lưu trong: localStorage.stoData.invest
     - Điều kiện bật Next: Nút Next chỉ bật khi người dùng đã chọn một option.
     - Logic khi nhấn Next
        - Mapping route tiếp theo:
          ```js
          nextRouter: [
            'revenue',
            'financialAssts',
            'asstsType',
            'experienceDerivative',
            'experienceFund',
            'desiredInvestmentType'
          ]
          ```
        - Khi nhấn Next:
          - Luồng đăng ký mới:
            |#|Màn hình hiện tại     |	Route tiếp theo      |
            |-|----------------------|-----------------------|
            |1|/annualIncome         |	/revenue             |
            |2|/revenue	             |/financialAssts        |
            |3|/financialAssts	     | /asstsType            |
            |4|/experienceStock	     | /experienceDerivative |
            |5|/experienceDerivative |	/experienceFund      |
            |6|/experienceFund       | /desiredInvestmentType|
            - Lưu ý: sau /financialAssts, luồng đi qua /asstsType, là component riêng, trước khi tới kinh nghiệm cổ phiếu.
          - Luồng chỉnh sửa:
            - Nếu vào từ /confirmInfo:
              - Lưu dữ liệu mới.
              - Không đi tiếp theo nextRouter.
              - Quay lại /confirmInfo.   
     - Logic nút Back:
       - Nếu vào từ `/confirmInfo`: Hiển thị dialog xác nhận trước khi quay lại /confirmInfo.
       - Nếu là luồng bình thường: Quay về trang trước đó theo Mapping route trước đó:
          ```js
            backRouter: [
              'purpose',
              'annualIncome',
              'revenue',
              'asstsType',
              'experienceStock',
              'experienceDerivative'
            ]
          ```
   - Tóm tắt luồng:
     ```
      /purpose
        ↓
      /annualIncome
        ↓
      Chọn thu nhập hằng năm
        ↓
      Lưu invest.income
        ↓
      /revenue
        ↓
      Chọn loại thu nhập
        ↓
      Lưu invest.revenuetype
        ↓
      /financialAssts
        ↓
      Chọn mức tài sản
        ↓
      Lưu invest.asset
        ↓
      /asstsType
        ↓
      /experienceStock
        ↓
      /experienceDerivative
        ↓
      /experienceFund
        ↓
      /desiredInvestmentType
     ```
## 2.12 desiredInvestmentType.vue
   - Location: [desiredInvestmentType.vue](/mobile-registration/src/views/PersonalResiger/Invest/desiredInvestmentType.vue)
   - Main function:  là màn hình cho phép người dùng chọn các sản phẩm đầu tư mà họ quan tâm. Người dùng có thể chọn nhiều lựa chọn cùng lúc:
      |ID|	Sản phẩm đầu tư                     |
      |--|--------------------------------------|
      | 1|	Cổ phiếu hiện vật                   |
      | 2|	Quỹ đầu tư, trái phiếu, kim loại quý|
      | 3|	Phái sinh, ví dụ hợp đồng tương lai |
      | 4|	Đầu tư ủng hộ một người/vật cụ thể  |
      | 5|	Đầu tư có ý nghĩa xã hội            |
      | 6|	Đầu tư bất động sản                 |
      | 7|	Khác                                |
     - Danh sách nằm trong [commonData.investGoodsData](/mobile-registration/src/utils/common-data.js#L229-L239).
   - Logic frontend:
     - Khởi tạo và khôi phục dữ liệu:
       - getInitialData() trong :
         ```js
          beforeRouteEnter(to, from, next) {
            next((vm) => vm.getInitialData(from.path));
          }
         ```
         - Lưu route trước đó vào fromRouter.
         - Lấy tiêu đề màn hình.
         - Đọc dữ liệu đã lưu:
           ```js
            this.commonJs.getLocalLowerData('goods', 'invest');
           ```
         - Dữ liệu thuộc: `localStorage.stoData.invest.goods`.
         - Nếu có dữ liệu cũ:
           - Khôi phục các sản phẩm đã chọn vào selectType.
           - Khôi phục nội dung nhập thêm vào other_input.
           - Đánh dấu lại các checkbox đã chọn.
     - Chọn nhiều sản phẩm
       - Không dùng cSingleChoice vì đây là lựa chọn nhiều giá trị. Component tự xử lý click:
        ```js
          handelSelect(item) {
            item.isActive = !item.isActive;

            if ([6, 7].includes(item.id) && !item.isActive) {
              this.other_input[item.id - 6] = '';
            }

            this.selectType = this.selectData.filter((item) => {
              return item.isActive;
            });
          }
        ```
        - Logic:
          - Bật/tắt isActive của option.
          - Nếu bỏ chọn id == 6 hoặc id == 7, xóa nội dung nhập thêm tương ứng.
          - Tạo lại selectType chỉ gồm các option đang được chọn.
     - Nhập nội dung bổ sung
       - Khi chọn:
         - id == 6: Đầu tư bất động sản.
         - id == 7: Khác.
       - Component hiển thị thêm một ô nhập:
         ```vue
          v-if="(item.id == 6 && item.isActive)
          || (item.id == 7 && item.isActive)"
         ```
       - Dữ liệu được lưu vào:
         ```js
          other_input[0] // id 6
          other_input[1] // id 7
         ```
       - Mapping dựa trên: `item.id - 6`
       - Điều kiện bật Next yêu cầu phải nhập nội dung tương ứng:
         ```js
          if (
            this.selectType.map(e => e.id).includes(6)
            && !this.other_input[0]
          ) {
            return false;
          }

          if (
            this.selectType.map(e => e.id).includes(7)
            && !this.other_input[1]
          ) {
            return false;
          }
        ```
       - Vì vậy:
         - Chọn bất động sản thì phải nhập thông tin bổ sung.
         - Chọn “Khác” thì phải nhập nội dung khác.
         - Nếu bỏ chọn, nội dung tương ứng bị xóa.
     - Điều kiện bật Next:
       - Nút Next chỉ bật khi:
         - Có ít nhất một sản phẩm đầu tư được chọn.
         - Nếu chọn id == 6, phải nhập other_input[0].
         - Nếu chọn id == 7, phải nhập other_input[1].
     - Logic khi nhấn Next:
        ```js
          handelNext() {
            let obj = {
              other: this.other_input,
              goods: this.selectType
            };

            this.commonJs.saveLocalLowerData(
              'goods',
              obj,
              'invest'
            );

            if (this.fromRouter == '/confirmInfo') {
              return this.$router.push('/confirmInfo');
            }

            this.commonJs.toNext('/electronic', null, true);
          }
        ```
       - Dữ liệu lưu vào:`localStorage.stoData.invest.goods`.
       - Luồng đăng ký mới: Sau khi lưu, chuyển tới `/electronic`
       - Luồng chỉnh sửa. Nếu đi từ /confirmInfo:
         - Lưu dữ liệu mới.
         - Quay lại /confirmInfo.
         - Không chuyển sang /electronic.
     - Logic nút Back:
       - Nếu vào từ `/confirmInfo`: Hiển thị dialog xác nhận trước khi quay lại /confirmInfo.
       - Nếu là luồng bình thường: Quay về trang trước đó. 
   - Tóm tắt luồng:
      ```
        /experienceFund
          ↓
        /desiredInvestmentType
          ↓
        Chọn một hoặc nhiều sản phẩm đầu tư
          ↓
        Nếu chọn bất động sản hoặc "Khác":
        nhập thông tin bổ sung
          ↓
        Kiểm tra điều kiện Next
          ↓
        Lưu invest.goods
          ↓
        /electronic
      ```
## 2.13 electronic.vue
   - Location: [electronic.vue](/mobile-registration/src/views/PersonalResiger/Confirm/electronic.vue)
   - Main function: là màn hình hiển thị tài liệu điện tử cần đọc và đồng ý trước khi tiếp tục quy trình mở tài khoản.
     - Màn hình xử lý:
       - Tải tài liệu PDF.
       - Hiển thị PDF trong iframe.
       - Theo dõi người dùng đã đọc PDF.
       - Cho phép người dùng đánh dấu “Đồng ý”.
       - Chỉ bật Next sau khi đã đồng ý.
       - Lưu thông tin đọc và đồng ý vào local storage.
       - Chuyển sang bước nhập thông tin cá nhân tiếp theo.
     - Tài liệu được sử dụng là tài liệu có: `DOCUMENT_TYPE == 26`.
   - Logic frontend:
     - Khởi tạo màn hình
       - Trong created():
         ```js
          this.getInitialData();
          this.$f7.preloader.show();
          window.checkPDFStatus = this.checkPDFStatus;
         ```
       - getInitialData() thực hiện:
         - Lấy tiêu đề và tiến độ.
         - Gọi getPdfList() để tải danh sách tài liệu.
         - Đọc dữ liệu đồng ý cũ từ: `this.commonJs.getLocalData('consentData') || [];`
         - Tìm bản ghi có: `item.DOCUMENT_TYPE == 26`.
         - Khôi phục: 
           - READ_DT
           - CONFIRM_DT
           - VERSION
         - Nếu đã có cả thời gian đọc và thời gian xác nhận:
            ```js
            if (this.read_dt && this.confirm_dt) {
                this.isAgree = true;
              }
            ```
         - Khi đó, người dùng có thể tiếp tục mà không cần đồng ý lại.
     - Tải tài liệu PDF:
       - Hàm getPdfList() gọi API:   
          ```js
            this.axios.get('/register/electronic_document')
          ```
       - Sau khi nhận response:
          ```js
            let pdfList = res.data.DATA;
          ```
       - Component lọc tài liệu:
          ```js
            let arr = pdfList.filter((e) => {
              return e.DOCUMENT_TYPE == 26;
            });
          ```
       - Nếu tìm thấy tài liệu: `this.pdf = arr[0];`
       - Dữ liệu pdf được dùng để hiển thị: `{{ pdf.DOCUMENT_NAME }}`.
       - và tạo URL cho PDF viewer: 
          ```js
            /static/pdfjs/web/viewer.html
            ?route=setEmail
            &url={{ pdf.DOCUMENT_PDF_PATH }}
          ```
       - API backend đã tạo sẵn URL S3 tạm thời cho DOCUMENT_PDF_PATH.
       - Nếu API lỗi, component hiển thị dialog lỗi: `res.data.ERROR.MESSAGE`.
     - Hiển thị và theo dõi PDF
       - PDF được hiển thị trong:
          ```js
            <iframe
              v-if="pdf.DOCUMENT_PDF_PATH"
              :src="'/static/pdfjs/web/viewer.html?...'"
            />
          ```
       - Component đăng ký callback global:
          ```js
            window.checkPDFStatus = this.checkPDFStatus;
          ```
       - Khi PDF viewer báo trạng thái:
          ```js
            checkPDFStatus(pdfStatus) {
              this.$f7.preloader.hide();

              if (pdfStatus == 'success') {
                this.readPdf();
              }

              if (pdfStatus == 'error') {
                this.$cErrorDialog(...);
              }
            }
          ```
       - Khi PDF đọc thành công:
          ```js
            readPdf() {
                if (
                  !this.read_dt ||
                  this.readedVersion != this.pdf.VERSION
                ) {
                  this.readedVersion = this.pdf.VERSION;
                  this.read_dt = this.Moment(new Date())
                    .format('YYYY-MM-DD HH:mm:ss');
                }

                this.$refs.radioPdf.isRead = true;
              }
          ```
         - Logic này:
          - Ghi nhận thời điểm đọc PDF.
          - Nếu phiên bản PDF thay đổi, cập nhật lại read_dt.
          - Đánh dấu component đồng ý đã được đọc bằng:
            ```js
            this.$refs.radioPdf.isRead = true;
            ```
       - Logic đồng ý tài liệu;
         - Khi người dùng nhấn nút đồng ý:
            ```js
              handelAgree(item) {
                this.isAgree = item;
                this.confirm_dt = this.Moment(new Date())
                  .format('YYYY-MM-DD HH:mm:ss');
              }
            ```
         - `isAgree` được dùng để bật/tắt nút Next:
            ```js
              <cBackNextButton
                :disabled="!isAgree"
                @clickNext="handelNext"
                @clickBack="handelBack"
              />
            ```
         - Nếu chưa đồng ý: Next bị disable
         - Nếu đã đồng ý: Next được bật
     - Logic khi nhấn Next
       - Hàm `handelNext` của component thực hiện:
         - Đọc danh sách đồng ý cũ.
         - Xóa bản ghi cũ có `DOCUMENT_TYPE == 26`.
         - Thêm bản ghi tài liệu hiện tại.
         - Lưu vào: `localStorage.stoData.consentData`
         - Chuyển sang: `/personalInfo`.
       - Dữ liệu được lưu:
          ```js
            {
              SEQ_NO: 123,
              DOCUMENT_TYPE: 26,
              VERSION: 1,
              READ_DT: '2026-09-24 10:00:00',
              CONFIRM_DT: '2026-09-24 10:01:00'
            }
          ```
     - Hiển thị PDF ở cửa sổ/popup riêng
       - Khi người dùng nhấn:
          ```js
          {{ pdf.DOCUMENT_NAME }}を別ウィンドウで表示
          ```
          hàm openPdf() chạy:
          ```js
            openPdf() {
              this.$f7.popup.open('.popup-pdf');

              this.$refs.read_pdf.pdf_ini(
                {
                  agree: false,
                  path: this.pdf.DOCUMENT_PDF_PATH,
                  isAgree: this.isAgree
                },
                this.pdf
              );
            }
          ```
          Component mở popup và truyền tài liệu vào cAggreePdf.
       - Khi đóng popup:
        ```js
          closePdf() {
            this.$f7.popup.close('.popup-pdf');
          }
        ```
   - Các API sử dụng:
     - API lấy danh sách tài liệu điện tử
       - Location: [ElectronicController.php](/common-front-api/app/Http/Controllers/ElectronicController.php).
       - Method: `GET`.
       - Endpoint: `/register/electronic_document`.
       - Frontend gọi tại: `getPdfList()`.
       - Backend: `ElectronicController::electronicDocument()`.
       - API trả về danh sách tài liệu có các trường chính:
         ```js
          {
            "SEQ_NO": 123,
            "DOCUMENT_TYPE": 26,
            "VERSION": 1,
            "DOCUMENT_NAME": "...",
            "DOCUMENT_PDF_PATH": "https://temporary-s3-url...",
            "DOCUMENT_TEXT": "...",
            "BASE_D": "2026-09-24 ..."
          }
         ```
        - Frontend chỉ lấy tài liệu có: `DOCUMENT_TYPE == 26`
   - Tóm tắt luồng:
# Step 3:


