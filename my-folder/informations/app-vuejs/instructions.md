# 1. Project Overview
- `app-vuejs` (declared as `st_fund_app_vuejs` in `package.json`) serves as the front-end Single Page Application (SPA) for the real estate security token (ST) trading platform.
- This web application enables online investors to perform daily tasks such as logging in, searching for or viewing ST product details, placing buy/sell orders, depositing/withdrawing funds, and updating personal information.

# 2. Project Goals
- Digital Securities Trading (ST): Enables users to search for and view details of digitized real estate projects in both primary (new issuance) and secondary markets. It supports priority purchase registration or participation in pre-sale lotteries, while guiding users through a rigorous buy/sell order placement process that complies with legal regulations (including document verification, investment suitability assessments, and insider status checks).
- Account Management & Secure Authentication: Facilitates login, registration/updating of personal and corporate information, setting transaction PINs, authenticating securities accounts, and account recovery processes.
- Cash Flow & Asset Management: Provides functions to deposit funds, issue withdrawal instructions, track wallet/balance fluctuations, and view an asset overview.
- History Lookup & Legal Reports: Enables investors to look up order and trade execution histories, as well as records of profit distributions and principal repayments; view electronic reports; and download documents to assist with tax settlement.

# 3. Tech Stack
The tech stack for the `app-vuejs` project is based on Vue 2, derived from the 2021 `vue-cli` template. The specific technologies and libraries used include:
- Core Framework & UI Foundation:
  - Vue.js: 2.6.14 (Core UI Framework).
  - Vue Router: 3.5.3 (Routing + auth guard).
  - Vuex: 3.6.2 (Shared state management is used, but its scope is very limited; the application relies primarily on sessionStorage.).
  - Framework7 / Framework7-Vue: 6.3.7 / 5.7.14 (Mobile UI layer: handles dialogs, loading indicators, popups, and side panels.).

- Build Tooling & Code Quality:
  - Webpack: ^3.6.0 (Use three configuration files in `build/`: `dev`, `dev.testing`, and `prod`.)
  - Babel: ^6 (Use preset-env and stage-2 for transpiling ES2015+ and Vue JSX.)
  - ESLint: ^4.15.0 (Complies with the `standard` and `plugin:vue/essential` configurations, and automatically runs via `eslint-loader` during every build.)
  - Testing Harness: Jest (^22.0.4) for unit testing and Nightwatch + Selenium (^0.9.12) for E2E testing.

- Backend Communication:
  - axios: 0.24.0 (A shared HTTP client with an 80-second timeout communicates with three API systems: Passport, Common, and Fund API.)

- Data Processing & Function Library:
  - Charting & Document Viewing: chart.js (^2.9.4) and vue-chartjs (^3.5.1) for rendering price charts; vue-pdf (^4.2.0) integrating the bundled pdf.js viewer to display reports and documents.
  - Time & Calculation: moment (2.29.1) / moment-timezone, numeral (^2.0.6), and mathjs (^10.4.0) for formatting and calculating monetary amounts and unit counts.
  - Security & Encryption: crypto-js (^4.2.0) / sha256 (^0.2.0) used to encrypt passwords before sending via API; sanitize-html (^2.4.0) used to sanitize HTML data from the server.
  - Data validation: libphonenumber-js (^1.9.15), email-validator (^2.0.4).

- Monitoring & Debugging:
  - Sentry: @sentry/vue & @sentry/tracing (^6.19.2) are used for monitoring and collecting runtime errors.
  - vConsole: ^3.4.1 (Console displayed on the screen of a real mobile device in Dev/Testing environments).

* Note on the CDN Externals mechanism: core libraries such as Vue, Vue Router, Vuex, axios, moment, and Framework7 are not bundled into the application. Instead, they are declared in the Webpack `externals` configuration and loaded directly via CDN (jsDelivr) in the `index.html` file. Consequently, the actual version running in the browser depends on the `<script>` tag in `index.html` rather than solely on `package.json`.

# 4. Project Structure
- Main source code directory (src/):
  - `src/main.js`: The project's most critical file. It serves as the entry point responsible for initializing Sentry, registering a suite of shared components, injecting utility modules into `Vue.prototype`, and housing the entire Router Auth Guard logic (`router.beforeEach`).
  - `src/App.vue`: Root component: Framework7 framework, screen transition animations, Zendesk widget initialization, and call to the `/init` API.
  - `src/router/index.js`: ~1,600 lines, 170 routes, each route lazy-loaded via `require.ensure`.
  - `src/store/index.js`: A very concise Vuex store file (48 lines) that manages only error dialog configuration, unread notification status, and user information.
  - `src/views/Resiger/`: Contains 174 main screens (views) and 8 subdirectories. Note: The directory name "Resiger" stems from a past typo of the word "Register" but has been retained because the entire router references it. The `.vue` filenames in this directory follow the Screen Code Convention (e.g., `g2107.vue`, `l2200.vue`).
  - `src/views/Login/index.vue` & `src/views/Read/read-pdf.vue`: Main login screen (route /) and PDF reading screen.
  - `src/components/`: Contains 51 shared components, 45 of which are automatically registered as global components via `src/components/index.js`.
  - `src/utils/`: Contains 9 modules handling shared logic, such as the HTTP Client (httpRequest.js), general helper functions (common.js), master data (common-data.js), and a list of investment-related questions (contentItem.js).
  - `src/assets/`: `assets/css/`: Global CSS system (not using `<style scoped>`). The core design relies on `pc.css` (~70KB) and `sp.css` (~77KB) for PC and mobile, complemented by two override files: `expand_pc.css` and `expand_sp.css`. `assets/img` and icons: Contains 152 SVG files and 19 raster image files..

- Configuration & Build Directory:
 - `build/`: Contains Webpack 3 configuration files for three environments: dev, dev.testing, and prod.
 - `config/`: Defines the corresponding `process.env` environment variables (prod.env.js, dev.env.js, dev.testing.env.js, test.env.js) and configures the dev-server.
 - `static/pdfjs/`: The bundled `pdf.js` PDF viewer will be copied intact to the `dist/` directory during the build process. 
 - `test/`: Contains template files for unit tests (Jest) and E2E tests (Nightwatch).
 - `.gitlab-ci.yml`: File defining the CI/CD pipeline (incorporating specific configuration files for staging3 and staging4).
 - `.env`: File containing environment variables for the local machine (do not commit to Git; based on the `.env.dev` or `.env.testing` template).

# 5. Architecture
- Router Navigation & Authorization Architecture:
  - Lazy Loading / Code Splitting: The system comprises 170 routes defined in `src/router/index.js`. Each screen is lazy-loaded using `require.ensure`, splitting the code into independent JS chunks based on the specific screen identifier.
  - Centralized Auth Guard (`router.beforeEach`): Located in `src/main.js`, this serves as the central mechanism for checking login status (via `meta.requireAuth`) and handling user authorization (Individual vs. Corporate). It also enforces mandatory redirection for accounts that have not yet set up a PIN (m2100) or verified their securities account number (m2102).
  - Force Reload Mechanism (Financial Security): For transaction screens involving transfers, withdrawals, or buy/sell operations (e.g., g2107, g2200, j2200), Router Guard uses `window.location.href` to perform a full page reload (Hard Reload). This completely clears the client-side state, ensuring absolute security for financial transactions.

- State and Browser Memory Management (State Management):
  - Vuex Store (`src/store/index.js`) — Used very sparingly. Although the project integrates Vuex (version 3.6.2), the `src/store/index.js` file is only about 48 lines long and stores just four main state variables:
    - `apiErrorMsgConfig`: Configuration for displaying a shared error notification dialog (listened for and updated by the `<my-dialog>` component in `App.vue` via `commonJs.showAlert()`).
    - `noticeRead`: Flag indicating whether there are unread notifications.
    - `showInviteMenu`: Flag to show/hide the invite-a-friend menu.
    - `userInfo`: Detailed user information retrieved from the /user/info API.
  - A distinctive feature of these Vuex mutations is that the mutations for `noticeRead` and `showInviteMenu` do not receive values ​​from the passed payload in the standard Vuex manner; instead, they are designed to retrieve values ​​from `sessionStorage` before assigning them to the state.
  - `sessionStorage` — The backbone for data transfer between screens:
    - Characteristics: Data is automatically lost when the tab is closed, isolating the session to each independent tab. If the API returns a 401 Unauthorized error, the entire `sessionStorage` is cleared.
  - Key points:
    - `ACCESS_TOKEN`: Authentication token (the presence of this key serves as a login check flag).
    - `USER`: Contains user JSON information (`PERSON_CORP_TYPE` [individual/corporate], `IS_SECRET_CONFIGURED`, `IS_ACCOUNT_NUMBER_VERIFIED`) for use by the Auth Guard.
    - `INIT_INFO`: Initialization data from the `/init` API (containing the number of funds `FUND_COUNT`, etc.).
    - `secretToken`: Token confirming the authentication of the transaction PIN.
    - `BUY_FUND_CD`, `FUND_NM`, `ORDERINFO`: Fund code, fund name, and order information passed between screens within the order placement flow.
    - `prevPageRoute`: Stores the route of the previous screen to facilitate branching logic.
    - `PAGE_AUTO_RELOAD`: Flag indicating a page transition caused by a hard reload.
  - `localStorage` — Persistent device storage; `localStorage` is used for information that needs to be retained over the long term:
    - `uuid`: A device identifier that is always included in all API requests via the `X-DU` HTTP header. When a user logs out, the system executes `localStorage.clear()` but always restores this `uuid` value.
    - `USERNAME`: Stores the remembered login ID.
    - `stoData`: Persistent storage data for specific screens.
  - Impact of the Force Reload (Hard Reload) mechanism on the state:
    - When navigating to screens involving financial transactions (such as buying/selling securities or depositing/withdrawing funds—e.g., g2107, g2200, j2200), Router Guard uses `window.location.href` to perform a full browser reload (Hard Reload).
    -This mechanism resets the entire Vuex state, preserving only the data stored in `sessionStorage`. This is an intentional design choice to avoid carrying over stale client-side state, not a system error.

- Backend API Connectivity Architecture:
  - The application communicates with three independent backend systems via URLs defined in environment variables:
    - `passportApiUrl`: Handles login and token issuance.
    - `commonApiUrl`: Handles user information, settings, deposits/withdrawals, and /init.
    - `fundApiUrl`: Handles subscription/issuance data, order placement, and transaction history.

  - Shared HTTP Client (src/utils/httpRequest.js): The project initializes a shared Axios instance in `src/utils/httpRequest.js` and injects it as `Vue.prototype.axios`, allowing any component to access it via `this.axios` with a fixed timeout of 80 seconds. Request Interceptor (Automatic Header Injection): Every outgoing request automatically includes the required custom headers:
    - `X-CI`: Company identifier (derived from the XCompanyId environment variable).
    - `X-DU`: Device identifier (retrieved from the UUID in localStorage).
    - `X-SV`, `X-PF`, `X-UA`, `X-MD`, `X-OV`: Information regarding app version, platform (iOS / Android / Web), User-Agent, device model, and OS version.
    - `Authorization`: The authentication token string `ACCESS_TOKEN` comes directly from `sessionStorage`.
    - `X-CT`: Reserved for financial transaction APIs (buy/sell, deposit/withdraw—located in the `transactionApi` array within `httpRequest.js`); it automatically retrieves the `ct` cookie value from Nginx (skipped in the VueJS dev environment where `VueJsJSEnv === 'dev'`).

  - Response Format & Error Handling: All system APIs return a consistent JSON structure:
    ```json
    {
    "STATUS": "OK" | "NG" | "MAINTENANCE",
    "DATA": { ... },
    "ERROR": { "CODE": "E100-0011", "MESSAGE": "..." }
    }
    ```
   * (Note: Even if the HTTP status is 200 OK, developers must still check the STATUS and ERROR fields in the body.).

  - Response Interceptor (Automatic error handling): The Response Interceptor automatically intercepts and handles special cases, eliminating the need to rewrite logic for individual screens:
    - Maintenance (STATUS === "MAINTENANCE"): Save the maintenance URL and navigate the application to the z1100 maintenance screen.
    - Non-JSON response / Missing STATUS field: Automatically redirects to the z1200 system error screen.
    - Account/device not verified (Error codes E100-0011, E100-0013, no_account_number_verified_error, xdu_credible_error): Display an alert to the user and redirect to the homepage.
    - HTTP 401 Unauthorized error: Clear all data in sessionStorage and localStorage (while retaining the UUID) and redirect to the login page (/).
    - HTTP 500 Error or Connection Loss: Automatic redirection to the z1200 system error screen.

  - Data Handling & Security:
    - Encryption of Passwords & Sensitive Information: Login and transaction passwords must never be transmitted in plain text. The `getEncryptionPassword()` function in `utils/common-api.js` automatically calls the `/common/encryption_key` API to retrieve the public key from the server and uses `crypto-js` to encrypt the password—along with the key version ID—before sending it via the API.
    - HTML Sanitization (XSS Sanitization): When displaying HTML data returned from the server via the `v-html` directive, you must pass the data through `this.$sanitize(data)` (using the `sanitize-html` library) to prevent XSS attacks.
    - Currency & Date Formatting: The project pre-injects `this.$Moment` (Moment.js) for date handling and provides a suite of helper functions within `this.commonJs` to convert between full-width and half-width characters, format currency with commas, and calculate unit quantities.

  - Shared API Module (src/utils/common-api.js): Instead of writing API calls directly within individual components, API operations used across multiple screens are consolidated in src/utils/common-api.js:
    - `getInit()`: Retrieves initial configuration information from /init.
    - `getUserInfo()`: Calls the `/user/info` API to retrieve the investor's latest information and automatically updates the Vuex store.
    - `getEncryptionPassword()`: Executes the process of retrieving the key and encrypting the password.
    - `checkJpText()`: Validates the character string (prevents the entry of Kanji into fields requiring Katakana).
    - `getFundIcon()`: Retrieves the display icon for each fund/issuance code.

- Automatic Component Registration & Utility Injection (Framework Infrastructure)
  - Global Components: 45 out of 51 shared components located in `src/components/` are automatically registered as global components in `src/main.js`.
  - Prototype Injection: Core utility modules are injected directly into `Vue.prototype` in `src/main.js` (e.g., `this.commonJs`, `this.commonData`, `this.contentItem`, `this.axios`, `this.$sanitize`, `this.$Moment`), allowing every component to access them directly without needing to re-import them.
  - Page Skeleton: Most screens follow a unified HTML tag structure featuring two distinct navigation components: one for PC (<pcnavi>) and one for mobile (<spnavi>).

- Style Management (Global Styling Architecture):
  - CSS is managed via centralized global stylesheets (without using `<style scoped></style>`).
  - The interface framework relies on two main files—`pc.css` (~70KB) for desktop and `sp.css` (~77KB) for mobile (with a breakpoint at 768px)—combined with `expand_pc.css` and `expand_sp.css` to override or supplement the interface styling.

# 6. Coding Conventions
- Enforced Rules: Defined in `.editorconfig` and `.eslintrc.js`, automatically checked via `eslint-loader` during every build.
  - Basic formatting: 2-space indentation, LF line endings, UTF-8 encoding, and automatic removal of trailing whitespace.
  - Strings: Single quotes '...' are mandatory (quotes: ['error', 'single']).
  - Linting ruleset: Complies with `standard` + `plugin:vue/essential` + `eslint:recommended`.
  - Variables & Debugging: Declaring unused variables is treated as an error (except for function parameters). `console` and `debugger` statements are considered errors in production builds. Disable checks for full-width spaces to allow comments in Japanese.
- Screen and File Naming Conventions:
  - Screen code: Set according to the formula [Business domain code letter] + [4 digits] (e.g., g2107, l2200).
  - File Name & Route:
    - View Components are located in the `src/views/Resiger/` directory. File names are written in lowercase, corresponding to the screen code (e.g., g2107.vue, l2300.vue).
    - Both the route path and route name use the screen code: /g2107. 
    - This screen codename is used consistently across both tickets (Backlog/Jira) and design documentation.
  - Shared Components:
    - The file name is located in `src/components/` and starts with the letter 'c' in camelCase (e.g., cTextInput.vue).
    - Global registration automatically capitalizes the first letter (CTextInput), allowing it to be invoked in templates as `<cTextInput>` or `<c-text-input>`.
- Practical conventions within the codebase:
  - Screen navigation: Typically invoked via the helper `this.commonJs.toNext('/l2100')` instead of using `this.$router.push` directly.
  - API calls: You must use the shared `this.axios` instance (injected into the Vue prototype); do not import axios directly within individual `.vue` files.
  - Language/Text Management: User notification messages should preferably be defined in the `src/utils/custom_language_jp.js` dictionary rather than hard-coded in the templates.
  - Commenting conventions: The codebase contains a mix of Japanese, Chinese, and English (due to the involvement of multiple vendors). New comments must be written in Japanese; however, existing Chinese comments should not be arbitrarily deleted or re-translated, so as to preserve the original author's intent.
  - Individual/Corporate Correspondence Rule: Many business workflows utilize separate screens for individual and corporate customers (e.g., l2200 for individuals vs. l2900 for corporate clients). When making modifications, developers are required to check and update the corresponding screens.
- Branch and Commit Message Conventions:
  - Branch Name: Follow the format `backlog/<TICKET_ID>` (e.g., `backlog/HASHST_DEV-4400`).
  - Commit Message: Place the Ticket ID and ticket title at the beginning, followed by a detailed description in Japanese.

# 7. Component Guidelines
- Component Classification and Naming Conventions.
  - View Components (184 files): Located in `src/views/Resiger/`, representing individual screens. File names are written in lowercase based on screen codes (e.g., `g2107.vue`, `l2300.vue`).
  - Shared Components (51 files): Located in `src/components/`, containing shared UI components. File names begin with the letter 'c' and follow the camelCase convention (e.g., `cTextInput.vue`, `cSinglePopup.vue`).
- Component Registration Mechanism (Registration Rules):
  - Automatic Global Registration: The `src/components/index.js` file exports 45 shared components. The `src/main.js` file automatically iterates through this list, capitalizes the first letter (e.g., `cTextInput` $\rightarrow$ `CTextInput`), and registers them as global components. In the template of any screen, you can invoke them directly as `<cTextInput>` or `<c-text-input>` without needing to import them again.
  - Mandatory requirement when creating a new shared component: If you create a new shared component in `src/components/`, you must declare its import and export in `src/components/index.js`; otherwise, the component will not be globally registered.
  - Local Components: There are six components in `src/components/` that are not globally registered (e.g., `cMulChioce.vue`, `rollerNumber.vue`). To use them, you must import them directly into the corresponding `.vue` file.
- Standard Page Skeleton Convention:
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
- Note on PC & Mobile interfaces: `<pcnavi>` and `<spnavi>` are separate components that toggle visibility based on a CSS media query (768px breakpoint). When modifying the menu or navigation, developers must update both components. The project also includes menu-hidden variants, such as `pcnaviMnone` and `spnaviMnone`.

- Core Shared Components (Key Input Components):
  - Basic data input: cTextInput, cTextInputNotSpace, cNumInput, cDecimalInput (with built-in validation).
  - Selectors (Select / Radio): cSelect, cDropDown, cRadio (or cDRadio from Input/radio/cRadio.vue), cCheckBox.
  - Popup / Modal: cSinglePopup, cSearchPopup, cPostalPopup (search for address by postal code), cSingleZonePopup.
  - Picker Mobile: scrollerPicker, inputPicker, datePicker (mobile-style roller picker).
  - Composite input blocks: cAddress, cNameBirthdaySex, and four components for declaring insider information (cInsiderFlg, cInsiderInfo, cInsiderAdd, cInsiderSeparation).
  - Notifications & Errors: myDialog (shared notification dialog), cErrors, cTips, Cnotice.
  - Buttons: Button, cRound Button, cNextOrBack Button (use the 'disable' class to hide or disable them).

- Technical guidelines to keep in mind when writing components:
  - Do not use Scoped CSS: The project makes virtually no use of `<style scoped></>`. All CSS is contained within four global stylesheets (`pc.css`, `sp.css`, `expand_pc.css`, `expand_sp.css`). Avoid creating duplicate class names within components; if UI adjustments are required, add a new class to `expand_pc.css` or `expand_sp.css`.
  - Leverage Injected Utilities: Do not re-import utility modules. Call them directly via the prototype: `this.axios`, `this.commonJs`, `this.commonData`, `this.contentItem`.
  - Data safety with `v-html`: If a component renders an HTML string from the server using the `v-html` directive, it is mandatory to wrap it with `this.$sanitize()` to prevent XSS security vulnerabilities (e.g., `v-html="$sanitize(content)"`).
  - Disable the loading effect (Framework7 Preloader): The loading display state is managed by Framework7 (`this.$f7.preloader.show()` and `this.$f7.preloader.hide()`). Always ensure that `.hide()` is called within API error-handling blocks to prevent the screen from freezing.

# 8. Testing
- Exercise extreme caution with financial transaction screens: When modifying cash flow screens (such as stock buy/sell or fund deposit/withdrawal functions like g2107, g2200, j2200, etc.), developers are required to conduct thorough manual testing.
- Testing on both account types (Individual & Corporate) is mandatory: Since many processes feature separate screens for Individual (PERSON_CORP_TYPE = 1) and Corporate (PERSON_CORP_TYPE = 2) customers (e.g., l2200 vs. l2900), it is common to test one side while overlooking the other.
- Testing on both interfaces (Desktop PC and Mobile Smartphone) is mandatory: The PC and mobile interfaces utilize completely independent navigation components (pcnavi/spnavi) and CSS files (pc.css/sp.css); consequently, making changes for one screen can easily break the layout on the other device.

# 9. Forbidden / Dangerous Changes
- Strict Data Security & Leak Prevention
  - Do not use `v-html` directly: You must wrap the data with `this.$sanitize()` to prevent XSS attacks.
  - Do not send passwords/PINs in plain text: You must use the `getEncryptionPassword()` helper in `common-api.js` to retrieve the encryption key from the server before transmission.

- Dangerous Source Code & Identifier Changes
  - Do not arbitrarily rename existing identifiers (HASHST, hashdash, hhd-): Although these names originate from the old brand (Hash DasH / Hash ST), they currently serve as actual environment variables and service names within AWS ECS and CI/CD workflows. Renaming them could cause system outages or break deployment pipelines.
  - Do not correct the misspelled directory name `src/views/Resiger/`: Although the name is a misspelling of "Register," all 170 routes in `src/router/index.js` reference it directly.
  -Do not arbitrarily rename existing CSS classes in `pc.css` or `sp.css`. Since the project rarely uses `<style scoped></style>`, the CSS is global in scope; renaming an existing class can easily break the layout on other screens. When modifications are required, add a new class to `expand_pc.css` or `expand_sp.css` instead.
  - Do not arbitrarily upgrade libraries to Vue 3: The project runs on the legacy Vue 2 platform (which has reached End-of-Life) with Webpack 3 and Babel 6. Compatibility with Vue 2 must be thoroughly verified before adding any dependencies.

- Prohibitions regarding Git and CI/CD:
  - Do not commit the .env file to Git: The .env file contains local configuration details and is listed in .gitignore. Under no circumstances should this file be pushed to the repository.
  - Direct pushes to the `master` branch are prohibited: All changes intended for `master` must be submitted via a Merge Request (MR) for review.
  - Avoid "silent" pushes to testing branches: Pushing code to branches such as `testing`, `testing_ph2`, or `testing3` triggers the CI/CD pipeline to immediately deploy to the shared test server, which may disrupt testing activities by other team members.

# 10. Important References
- Reference Document Status in Repository:
  - Interface reference files (sample1 ~ sample10): Located in `src/views/Resiger/`, these are pre-built screen templates. While these files do not run in the production environment, they serve as structural references when creating new screens.
- Glossary:
  - Financial operations: ST (Digital Securities), Primary/Secondary, Priority application/Lottery, Order matching, Number of trading units, Suitability questionnaire, eKYC (Electronic Know Your Customer), Transaction PIN.
  - Technical: SPA, Routing/Route Guard, Code Splitting/Lazy Loading, Interceptor, sessionStorage/localStorage, ECS/ECR/CodeDeploy, Blue/Green Deployment.
