# Admin Back-Office SPA — Bundle Reverse-Engineering Report

**Target:** `src-analysis/admin/index.js` (8.7 MB, single-file production JS bundle)
**Build tooling:** Vite + Rollup, `MODE:"production"`
**API host:** `https://white-label-adminapi.gammaplus.io`
**Method:** Static analysis only (grep/regex over minified source + targeted Python extraction). No requests were made against the live API.

---

## 1. Tech Stack

Version strings are stripped by the minifier, so identification is based on library fingerprints (internal class/plugin names, error-message URLs, config option names, CSS class names).

| Category | Library | Evidence |
|---|---|---|
| UI framework | React (with `react-dom`, dev-mode `jsxDEV` calls present) | `x.jsxDEV(...)`, `fileName:"/...jsx"` debug annotations baked into prod build |
| State management | Redux + **redux-saga** (not thunk/RTK-query) | `redux-saga` string literal x7, generator-based effect helpers, `takeLatest`/`call`/`put`-style saga machinery |
| Routing | react-router (v6-style) | `react-router` x2 |
| HTTP client | **axios** (custom wrapper) | axios internals present (`AxiosHeaders`, `transformRequest`, `paramsSerializer`, `allowAbsoluteUrls`); app wraps it in 4 helper functions `Pr`=GET, `Ar`=POST, `W0`=PUT, `yg`=DELETE (confirmed via `gee(e,cee.get/post/put/delete,...)`) |
| Forms | **Formik** + **Yup** | `formik` x10, `yup` x28, `useFormikContext` present |
| UI kit / CSS | **Bootstrap** grid/utility classes (not reactstrap component library — 0 hits) | `col-md-12`, `btn-primary` classes hand-composed with `className` strings |
| Charts | **Apache ECharts** | `echarts` literal, ECharts-specific option keys (`axisExpandCenter`, `barCategoryGap`, `avoidLabelOverlap`, etc.) |
| Rich text editor | **CKEditor 5** (extensive plugin set) | Plugin classes: `BalloonToolbar`, `ClipboardPipeline`, `CKBoxEditing`, `ImageUploadEditing`, `TrackChangesEditing`, `WidgetToolbarRepository`; CKEditor error-docs URL string |
| Real-time | **socket.io-client** | `EIO=` engine.io handshake marker; app-level channel names `/private/disputeThreadUpdated`, `/private/suspiciousLogin`; `/api/socket` route |
| Error monitoring | **Sentry** (`@sentry/react` or similar) | `Sentry.init({...})`, `X-Sentry-Rate-Limits` header handling — **but `VITE_APP_SENTRY_DSN` is absent from the compiled env object in this build**, so no DSN ships in this artifact |
| Dates | **moment.js** | `moment` x64 |
| Date picker widget | **react-datepicker** | 186 hits — used pervasively across nearly every filter/form screen |
| i18n | **i18next** + **react-i18next** | `i18next.use(...).init({resources:{...}})`, `react-i18next` x2 |
| Utilities | **lodash** (`Hi` alias seen calling `.isEmpty`) | `lodash` x15 |
| Encryption | **CryptoJS** (AES) | `Dfe.AES.encrypt/decrypt` |
| Misc | select widgets (`react-select`), CKBox cloud image service integration | — |

**Not found:** reactstrap, MUI/@mui, react-bootstrap, react-hook-form, react-toastify/Toastify, recharts/Chart.js/Victory/Nivo, react-table/@tanstack/react-table, sweetalert2, redux-persist. Tables and toasts appear to be in-house components rather than a named third-party library.

Twelve UI languages are bundled: **EN, DE, ES, FI, FR, IT, JA, KO, NL, PT, RU, ZH**.

---

## 2. API Endpoint Inventory

### 2.1 How the client is structured

The app is organized as a set of "microservice"-style route namespaces prefixed onto a common base. A shared constant object (found verbatim in the bundle) defines the namespace-to-path-prefix mapping:

```js
rn = {
  CASINO: "/casino-management/",
  CONTENT: "/content-management/",
  PLAYER: "/player-management/",
  SETTINGS: "/settings/",
  LANGUAGE: "/language/",
  SPORTS: "/sportsbook-management/",
  ADMIN: "/admin/",
  COUNTRY: "/country/",
  REPORT: "/report/",
  TRANSACTION: "/transaction/",
  DASHBOARD: "/dashboard/",
  BONUS: "/bonus-management/",
  TOURNAMENT: "/tournament",
  PAYMENT: "/payment/",
  DISPUTE: "/dispute-management/",
  CHAT: "/live-chat/",
  INTERNAL: "/internal/",
  GAMIFICATION: "/gamification/",
  AI: "/ai/",
  WORKFLOW: "/workflow/"
}
```

Every call is built as `` `${API_URL}${API_VERSION}${rn.MODULE}<action>` `` where `API_VERSION` is the literal string `"/api/v2"`. A second, separate base (`Xx`) is used only for the newer analytics dashboard, calling `` `${Xx}/api/v3/dashboard-v2/<report>` `` directly (bypassing the `rn` namespace object) — this looks like a newer microservice added after the original v2 API.

The four HTTP verbs are confirmed by their wrapper-function bodies:

```js
Pr = (e,t) => gee(e, cee.get, {}, {}, t)      // GET
Ar = (e,t,n) => gee(e, cee.post, t, n)        // POST
W0 = (e,t,n) => gee(e, cee.put, t, n)         // PUT
yg = (e,t) => gee(e, cee.delete, t)           // DELETE
```

251 distinct calls were extracted this way (114 POST, 100 GET, 14 DELETE, 12 PUT, plus 11 GET-only calls against the separate v3 dashboard base). Full list below, grouped by module.

### 2.2 ADMIN (`/api/v2/admin/…`) — staff, roles, gallery, login

| Method | Path |
|---|---|
| POST | `admin/login` |
| POST | `admin/create-user` |
| POST | `admin/change-password` |
| POST | `admin/update-profile` |
| GET | `admin/staff` |
| GET | `admin/roles` |
| GET | `admin/children` |
| POST | `admin/toggle-child` |
| POST | `admin/update-child` |
| POST | `admin/update-site-layout` |
| GET | `admin/review` / POST `admin/review` / PUT `admin/review` |
| GET | `admin/gallery-images` |
| POST | `admin/upload-gallery-image` |
| POST | `admin/delete-gallery-image` |
| GET | `admin/gallery-folders` |
| GET | `admin/suspicious-login/count` |
| POST | `admin/update-suspicious-login/isRead` |
| GET | `/api/v2/admin` (bare, outside `rn`) |

### 2.3 PLAYER (`/api/v2/player-management/…`) — largest module, 45+ endpoints

Player CRUD/search: `player` (GET), `players` (GET), `player/update` (POST), `player/toggle` (POST), `player/reset-password` (POST), `player/update-password` (POST), `player/create-comment` / `player/update-comment` (POST) / `player/comment` (DELETE), `duplicate-players`, `duplicate-players/all`, `devices-report`, `devices-users`, `/ip-lookup`, `/login-history`, `wallet` (POST).

KYC sub-resource: `kyc/kyc-requests` (GET), `kyc/methods` (GET), `kyc/document-labels` (GET), `kyc/document-label/create` / `update` (POST), `kyc/activate`, `kyc/inactive`, `kyc/reject-document`, `kyc/request-document`, `kyc/verify-document`, `kyc/verify-email`, `kyc/toggle-kyc-method` (all POST).

Limits: `limit/update-betting`, `limit/update-deposit-and-loss`, `limit/update-self-exclusion` (POST — responsible-gambling controls).

Segmentation: `segments` (GET), `segment-users` (GET), `segments/constants` (GET), `segment` (POST create / PUT update / DELETE `segment?id=`), `segment-advance-filter` (POST).

Tags: `tags` (GET), `tag/create`, `tag/attach-tag`, `tag/update-tag`, `tag/remove-tag` (POST), `tag` (DELETE).

Suspicious activity: `suspicious-activities` (GET/POST), `suspicious-activities-settings` (GET/POST), `suspicious-player/toggle` (POST).

Notifications: `notification` (GET), `notification/create` (POST).

### 2.4 BONUS (`/api/v2/bonus-management/…`)

`bonus`, `bonuses`, `bonus-type-count` (GET); `bonus/create`, `bonus/update`, `bonus/toggle`, `bonus/reorder` (POST), `bonus/delete` (DELETE); `loyalty-levels` (GET), `loyalty-level/create`, `loyalty-level/update` (POST), `loyalty-level/delete` (DELETE); `wagering-template(s)` (GET), `wagering-template/update` (POST); `spin-wheel-config` (GET/update POST); `referral/transactions`, `referral/users` (GET), `referral/update` (POST). Plus outside-`rn` calls: `bonus/issue` (POST), `bonus/cancel` (PUT), `bonus/user` (GET).

### 2.5 CASINO (`/api/v2/casino-management/…`)

`games`, `providers`, `aggregators`, `categories` (GET); `edit-game`, `edit-provider`, `edit-aggregator`, `create-category`, `edit-category`, `toggle`, `toggle-featured-game`, `add-games-to-category`, `remove-games-from-category`, `reorder-category`, `reorder-games`, `reorder-provider`, `restrict-countries-for-game`, `restrict-countries-for-provider`, `remove-restricted-countries-for-game`, `remove-restricted-countries-for-provider` (all POST); `delete-category` (DELETE). Plus `casino/aggregator` (POST, outside `rn`).

### 2.6 CONTENT / CMS (`/api/v2/content-management/…`)

Pages: `pages`, `page?pageId=` (GET); `page/create`, `page/update`, `page/toggle` (POST); `page` (DELETE).
Banners: `banner`, `banner/types` (GET); `banner/create`, `banner/update`, `banner/reorder` (POST); `banner` (DELETE).
Email templates: `email-templates`, `email-template?emailTemplateId=` (GET); `email/create`, `email/update`, `email/set-default` (POST); dynamic-data at `/api/v2/cms/dynamic-data`, `/api/v2/email/dynamic-data`, `/api/v2/email/test` (outside `rn`).
Gallery: `gallery` (GET/DELETE), `gallery/upload` (POST).

### 2.7 SETTINGS (`/api/v2/settings/…`)

`application`, `countries`, `currencies`, `languages`, `global-registration` (GET); `application/toggle`, `application/update-chat-setting`, `application/update-constants`, `application/update-logo`, `application/update-value-comparisons` (POST); `country/toggle`, `country/update` (POST); `currency/create`, `currency/toggle`, `currency/update` (POST); `/api/v2/global-registration` (PUT, outside `rn`).

### 2.8 SPORTS (`/api/v2/sportsbook-management/…`) — sportsbook back-office

`sports`, `events`, `leagues`, `locations`, `markets`, `match-markets?matchId=`, `bet-settings` (GET); `toggle-sport`, `toggle-location`, `upload-sport-icon`, `upload-location-icon` (POST). Plus non-`rn` calls: `sportsbook/bet-settings` (POST+PUT), `sportsbook/custom-odds`, `sportsbook/detach-market`, `sportsbook/odd-settings` (PUT) — these look like they target a separate sportsbook microservice reached via base `J0`.

### 2.9 TRANSACTION / REPORT (`/api/v2/transaction/…`)

`banking-transactions`, `betslip-report`, `casino-transactions`, `sportsbook-transactions`, `get-ledgers`, `report` (GET, and `report` also POST for exports/downloads).

### 2.10 DASHBOARD (legacy v2 + new v3)

v2: `dashboard/get-kpi-summary`, `get-kpi-report`, `get-demograph`, `get-game-report`, `player-performance-sapshot` [sic], `statistics-summary`.

v3 (separate host/base `Xx`, likely a newer analytics microservice): `/api/v3/dashboard-v2/casino-dashboard`, `casino-heat-map`, `casino-session-length`, `game-report`, `bet-distribution`, `bonus-summary`, `bonus-expiry-trend`, `bonus-liability-wager-volume`, `top-bonus-players-info`, `financial-data`, `financial-daily-report`.

### 2.11 TOURNAMENT (`/api/v2/tournament/…`)

`tournament` (GET list), `tournament/details`, `tournament-leaderBoard`, `tournament-transactions` (GET); `create`, `update`, `toggle`, `cancel`, `settlement` (POST).

### 2.12 CHAT / live-chat (`/api/v2/live-chat/…`)

`get-group`, `get-group-chats`, `get-group-users`, `get-chat-rain`, `get-offensive-words`, `messages/messages-group` (GET); `create-group`, `create-chat-rain`, `create-offensive-word`, `update-chat-rain` (POST); `delete-group`, `delete-chat-rain`, `delete-offensive-word` (DELETE); `ban-group-user`, `update-offensive-word` (PUT); `live-chat/update-group` (POST, outside `rn`); `/api/v2/all-group` (GET).

### 2.13 DISPUTE (`/api/v2/dispute-management/…`, actually served under `crm/dispute-management/…`)

`get-ticket`, `get-ticket-details` (GET); `reply-message`, `update-status` (POST). Real-time push via socket event `/private/disputeThreadUpdated`.

### 2.14 GAMIFICATION (`/api/v2/gamification/…`)

`tasks`, `task-details` (GET); `create-task`, `update-task`, `toggle` (POST); `delete-task` (DELETE).

### 2.15 AI (`/api/v2/ai/…`)

`ai-chat-history` (GET), `chat-bot` (POST) — an in-admin AI chatbot feature.

### 2.16 INTERNAL / WORKFLOW / COUNTRY / LANGUAGE

- INTERNAL: `credentials` (GET), `create-credentials`/`update-credentials` (POST) — third-party integration credentials (payment/game aggregator API keys), plus `/api/v2/user/internal` (PUT) and `/user/all-withdraw-request` (GET).
- COUNTRY: `restricted-items`, `unrestricted-items` (GET/DELETE), `restricted` (PUT).
- LANGUAGE: `support-keys?language=` — i18n key export/download endpoint (also seen with `&csvDownload=true&token=`).

Full raw endpoint table (auto-extracted, 251 rows) is preserved for reference below the fold in the analysis scratch data if deeper drill-down is ever needed; the above groups cover it exhaustively by module.

---

## 3. Redux Action Types (domain model)

268 `..._LOADING` / `_SUCCESS` / `_FAILURE` / `_REQUEST` / `_ERROR` action-type constants were extracted. Grouped by feature area:

| Domain | Representative actions |
|---|---|
| **Auth/Admin** | `LOGIN_SUCCESS`, `LOGIN_API_ERROR`, `LOGOUT_USER_SUCCESS`, `UPDATE_PROFILE_SUCCESS`, `RESET_PROFILE_PASSWORD_SUCCESS`, `GET_ADMINS_DATA_SUCCESS`, `GET_ADMIN_CHILDREN_SUCCESS`, `ADD_SUPER_ADMIN_USER_SUCCESS`, `UPDATE_SUPER_ADMIN_USER_SUCCESS`, `UPDATE_SUPER_ADMIN_STATUS_SUCCESS`, `SUPER_ADMIN_SUCCESS`, `ROLES_SUCCESS`/`ROLES_ERROR`, `GET_STAFF_ADMIN_BY_ID_SUCCESS/FAILURE`, `UPDATE_SA_USER_STATUS_SUCCESS` |
| **Player** | `FETCH_PLAYERS_SUCCESS`, `GET_USER_DETAILS_SUCCESS`, `UPDATE_USER_INFO_SUCCESS`, `UPDATE_USER_PASSWORD_SUCCESS`, `BAN_PLAYER_SUCCESS/FAILURE`, `DISABLE_USER_SUCCESS`, `MARK_USER_AS_INTERNAL_SUCCESS`, `VERIFY_USER_EMAIL_SUCCESS`, `GET_DUPLICATE_USERS_SUCCESS`, `FETCH_USER_DEVICE_HISTORY_SUCCESS/FAILURE`, `FETCH_PLAYERS_LOGIN_DEVICE_AND_SESSION_SUCCESS`, `FETCH_IP_LOOKUP_SUCCESS/LOADING` |
| **KYC** | `ACTIVATE_KYC_SUCCESS`, `INACTIVE_KYC_SUCCESS`, `REJECT_DOCUMENT_SUCCESS`, `REQUEST_DOCUMENT_SUCCESS`, `VERIFY_DOCUMENT_SUCCESS`, `GET_KYC_REQUESTS_SUCCESS`, `GET_KYC_SETTINGS_SUCCESS`/`KYC_SETTINGS_SUCCESS`, `GET_PENDING_KYC_COUNT_SUCCESS`, `EDIT_KYC_LABELS_SUCCESS`, `CREATE_KYC_LABELS_SUCCESS`, `GET_DOCUMENT_LABEL_SUCCESS`, `GET_USER_DOCUMENTS_SUCCESS` |
| **Bonus/Loyalty** | `CREATE_BONUS_SUCCESS`, `UPDATE_BONUS_SUCCESS`, `REORDER_BONUS_SUCCESS`, `UPDATE_SA_BONUS_STATUS_SUCCESS`, `ISSUE_BONUS_SUCCESS`, `GET_BONUS_DETAILS_SUCCESS`, `GET_ALL_BONUS_SUCCESS`, `GET_ALL_BONUSES_TYPE_SUCCESS`, `CREATE_LOYALTY_LEVEL_SUCCESS`, `DELETE_LOYALTY_LEVEL_SUCCESS`, `GET_LOYALTY_LEVEL_SUCCESS`, `GET_SPIN_WHEEL_DATA_SUCCESS`, `UPDATE_SPIN_WHEEL_SUCCESS`, `RESET_SPIN_WHEEL_SUCCESS`, `CREATE_WAGERING_TEMPLATE_DETAILS_SUCCESS`, `EDIT_WAGERING_TEMPLATE_DETAILS_SUCCESS`, `GET_WAGERING_TEMPLATE_DETAIL(S)_SUCCESS`, `EDIT_ALL_REFERRALS_SUCCESS`, `FETCH_ALL_REFERRALS_SUCCESS`, `CANCEL_USER_BONUS_SUCCESS`, `GET_USER_BONUS_SUCCESS` |
| **Casino content** | `CREATE_CASINO_CATEGORY_SUCCESS`, `EDIT_CASINO_CATEGORY_SUCCESS`, `DELETE_CASINO_CATEGORY_SUCCESS`, `REORDER_CASINO_CATEGORY_SUCCESS`, `EDIT_CASINO_GAMES_SUCCESS`, `REORDER_CASINO_GAMES_SUCCESS`, `GET_CASINO_GAMES_SUCCESS`, `ADD_GAME_TO_CASINO_CATEGORY_SUCCESS`, `REMOVE_GAME_FROM_CATEGORY_SUCCESS`, `GET_ADDED_GAMES_IN_CATEGORY_SUCCESS`, `UPDATE_GAME_ISFEATURED_SUCCESS`, `CREATE_CASINO_PROVIDERS_SUCCESS`, `EDIT_CASINO_PROVIDERS_SUCCESS`, `REORDER_CASINO_PROVIDER_SUCCESS`, `GET_CASINO_PROVIDERS_DATA_SUCCESS`, `CREATE_AGGREGATORS_SUCCESS`, `UPDATE_AGGREGATORS_SUCCESS/STATUS_SUCCESS`, `GET_AGGREGATORS_SUCCESS/FAILURE`, `ADD_RESTRICTED_COUNTRIES_SUCCESS`, `REMOVE_RESTRICTED_ITEMS_SUCCESS`, `FETCH_RESTRICTED_ITEMS_SUCCESS`, `FETCH_UNRESTRICTED_ITEMS_SUCCESS`, `UPDATE_CASINO_STATUS_SUCCESS` |
| **Payments** | `ADD_PAYMENT_SUCCESS`, `CREATE_PAYMENT_SUCCESS`, `UPDATE_PAYMENT_SUCCESS`, `UPDATE_PAYMENT_CREDENTIALS_SUCCESS`, `GET_PAYMENT_DATA_SUCCESS`, `GET_PAYMENT_DETAILS_SUCCESS`, `GET_PAYMENT_PROVIDER_DATA_SUCCESS`, `GET_OFAPAY_CURRENCIES_SUCCESS/FAILURE`, `GET_OFAPAY_METHODS_SUCCESS`, `GET_OFAPAY_BANKS_SUCCESS`, `FETCH_WITHDRAW_REQUESTS_SUCCESS`, `DEPOSIT_TO_OTHER_SUCCESS` |
| **Transactions/Reports** | `FETCH_CASINO_TRANSACTIONS_SUCCESS`, `FETCH_SPORTS_TRANSACTIONS_SUCCESS`, `FETCH_SPORTS_BET_SUCCESS`, `FETCH_GAME_TRANSACTIONS_SUCCESS`, `FETCH_TRANSACTION_BANKING_SUCCESS`, `GET_LEDGER_DETAILS_SUCCESS`, `DOWNLOAD_REPORT_SUCCESS/FAILURE`, `FETCH_PLAYER_DEVICE_REPORT_SUCCESS`, `FETCH_PLAYER_PERFORMANCE_SUCCESS`, `GET_GAME_REPORT_SUCCESS`, `GET_KPI_REPORT_SUCCESS`, `GET_KPI_SUMMARY_SUCCESS`, `GET_DEMOGRAPHIC_SUCCESS`, `GET_STATS_SUCCESS`, `GET_TOP_PLAYERS_SUCCESS` |
| **Tournament** | `CREATE_TOURNAMENT_SUCCESS`, `UPDATE_TOURNAMENT_SUCCESS`, `UPDATE_TOURNAMENT_STATUS_SUCCESS`, `GET_TOURNAMENT_DETAILS_SUCCESS`, `GET_TOURNAMENT_DETAIL_BY_ID_SUCCESS`, `GET_TOURNAMENT_LEADERBOARD_DETAIL_SUCCESS`, `GET_TOURNAMENT_TRANSACTIONS_SUCCESS`, `FETCH_SPORTS_TOURNAMENT_LIST_SUCCESS` (naming overlap — sports tournaments vs casino tournaments share reducer namespace) |
| **Sportsbook** | `GET_SPORTS_LIST_SUCCESS`, `GET_SPORTS_COUNTRIES_SUCCESS`, `GET_SPORTS_MATCHESDETAIL_SUCCESS/FAILURE`, `FETCH_SPORTS_MARKETS_SUCCESS`, `FETCH_SPORTS_MATCHES_SUCCESS`, `UPDATE_SPORTS_FEATURED_MATCHES_SUCCESS`, `CREATE_BET_SETTINGS_SUCCESS`, `EDIT_BET_SETTINGS_SUCCESS`, `GET_BET_SETTINGS_DATA_SUCCESS`, `UPDATE_COMPANYODD_SUCCESS/FAILURE`, `DETACH_ODDSVARIATION_SUCCESS/FAILURE`, `UPDATE_ODDSVARIATION_SUCCESS/FAILURE`, `UPLOAD_SPORTS_COUNTRY_IMAGE_SUCCESS/FAILURE` |
| **CMS/Content** | `CREATE_SA_CMS_SUCCESS`, `UPDATE_SA_CMS_SUCCESS/STATUS_SUCCESS`, `DELETE_CMS_SUCCESS`, `GET_ALL_CMS_DATA_SUCCESS`, `GET_CMS_BY_PAGE_ID_SUCCESS`, `GET_CMS_DYNAMIC_KEYS_SUCCESS`, `GET_DYNAMIC_KEYS_SUCCESS`, `CREATE_BANNERS_SUCCESS`, `DELETE_BANNERS_SUCCESS`, `EDIT_SA_BANNERS_SUCCESS`, `GET_SA_BANNERS_SUCCESS`, `REORDER_BANNER_SUCCESS`, `CREATE_EMAIL_TEMPLATE_SUCCESS`, `UPDATE_EMAIL_TEMPLATE_SUCCESS`, `DELETE_EMAIL_TEMPLATE_SUCCESS`, `GET_EMAIL_TEMPLATE(S)_SUCCESS`, `GET_EMAIL_TYPES_SUCCESS`, `MAKE_EMAIL_TEMPLATE_PRIMARY_SUCCESS`, `TEST_EMAIL_TEMPLATE_SUCCESS`, `CREATE_GALLERY_FOLDER_SUCCESS`, `UPDATE_GALLERY_FOLDER_SUCCESS`, `DELETE_GALLERY_FOLDER_SUCCESS`, `GET_GALLERY_FOLDERS_SUCCESS`, `GET_GALLERY_IMAGES_SUCCESS`, `UPLOAD_GALLERY_IMAGE_SUCCESS`, `DELETE_GALLERY_IMAGE_SUCCESS`, `UPLOAD_IMAGE_GALLERY_SUCCESS`, `DELETE_IMAGE_GALLERY_SUCCESS`, `UPLOAD_IMAGE_SUCCESS/FAILURE` |
| **Settings/Config** | `GET_SITE_CONFIGURATION_SUCCESS`, `FETCH_COUNTRIES_SUCCESS`, `EDIT_COUNTRIES_SUCCESS`, `UPDATE_COUNTRIES_STATUS_SUCCESS`, `FETCH_CURRENCIES_SUCCESS`, `CREATE_CURRENCIES_SUCCESS`, `EDIT_CURRENCIES_SUCCESS`, `FETCH_LANGUAGES_SUCCESS`, `FETCH_LANGUAGE_MANAGEMENT_SUCCESS`, `GET_LANGUAGE_DATA_SUCCESS`, `GET_REGISTRATION_FIELDS_SUCCESS`, `UPDATE_REGISTRATION_FIELDS_SUCCESS` |
| **Chat / Live-chat** | `CREATE_CHANNEL_SUCCESS`, `UPDATE_CHANNEL_DETAILS_SUCCESS/FAILURE`, `DELETE_CHANNEL_SUCCESS/FAILURE`, `GET_CHANNELS_SUCCESS`, `GET_CHANNEL_GROUP_SUCCESS/FAILURE/REQUEST`, `GET_CHANNEL_MESSAGES_SUCCESS`, `SEND_MESSAGE_SUCCESS`, `GET_GROUP_CHATS_SUCCESS/FAILURE/REQUEST`, `GET_ALL_GROUP_SUCCESS`, `UPDATE_CHAT_SETTINGS_SUCCESS`, `CREATE_CHATRAIN_SUCCESS`, `UPDATE_CHATRAIN_SUCCESS`, `DELETE_CHATRAIN_SUCCESS`, `GET_CHATRAIN_DATA_SUCCESS`, `CREATE_OFFENSIVE_WORDS_SUCCESS`, `EDIT_OFFENSIVE_WORDS_SUCCESS`, `FETCH_OFFENSIVE_WORDS_SUCCESS`, `CONNECT_ERROR` (socket.io) |
| **Disputes** | `FETCH_DISPUTES_SUCCESS`, `FETCH_DISPUTE_SUCCESS` |
| **Tags/Segments/Notes** | `CREATE_TAG_SUCCESS`, `UPDATE_TAG_SUCCESS`, `DELETE_TAG_SUCCESS`, `ATTACH_TAG_SUCCESS`, `REMOVE_TAG_SUCCESS`, `GET_ALL_TAGS_SUCCESS`, `CREATE_SEGMENTATION_SUCCESS`, `UPDATE_SEGMENTATION_SUCCESS`, `DELETE_SEGMENTATION_SUCCESS`, `FETCH_SEGMENTATION_SUCCESS/DETAILS_SUCCESS/CONSTANTS_SUCCESS`, `CREATE_USER_COMMENT_SUCCESS`, `UPDATE_USER_COMMENT_SUCCESS`, `DELETE_USER_COMMENT_SUCCESS`, `GET_USER_COMMENTS_SUCCESS` |
| **Suspicious activity / risk** | `FETCH_SUSPICIOUS_ACTIVITIES_SUCCESS`, `FETCH_SUSPICIOUS_ACTIVITY_SUCCESS`, `UPDATE_SUSPICIOUS_ACTIVITY_SUCCESS`, `RESET_SUSPICIOUS_ACTIVITY_UPDATE_SUCCESS`, `FETCH_SUSPICIOUS_ACTIVITIES_SETTINGS_SUCCESS`, `UPDATE_SUSPICIOUS_ACTIVITIES_SETTINGS_SUCCESS`, `UPDATE_SUSPICIOUS_PLAYER_STATUS_SUCCESS` |
| **Gamification** | `CREATE_GAMIFICATION_SUCCESS`, `UPDATE_GAMIFICATION_SUCCESS`, `UPDATE_SA_GAMIFICATION_STATUS_SUCCESS`, `GET_GAMIFICATIONS_SUCCESS`, `GET_GAMIFICATION_DETAIL_SUCCESS`, `GET_ALL_GAMIFICATIONS_TYPE_SUCCESS` |
| **Notifications** | `FETCH_NOTIFICATIONS_SUCCESS`, `NOTIFY_PLAYERS_SUCCESS`, `SEND_PASSWORD_RESET_SUCCESS` |
| **Generic/infra** | `API_SUCCESS`, `SUBMIT_SUCCESS/FAILURE`, `SET_FIELD_ERROR`, `APPLY_ADVANCE_FILTER_SUCCESS/FAILURE`, `ERR_BAD_REQUEST` (axios), `LOGIN_API_ERROR` |

---

## 4. Permissions / RBAC Model

The app implements **module-based, letter-coded CRUD permissions**, not fixed named roles. Two static constants define the model:

**Module registry** (`wt` object — every permission-gated feature area):

```js
wt = {
  ai, kyc, tag, page, admin, bonus, banner, gallery, limits, player, report,
  comment, country, currency, language, emailTemplate, casinoManagement,
  contentManagement, applicationSetting, sportsbookManagement,
  kpiSummaryReport, livePlayerDetail, gameReport, kpiReport, demography,
  review, tournamentManagement, paymentManagement, disputeManagement,
  segment, referral, gamification, chatManagement, suspiciousLoginNotification
}
```
(34 permission-gated modules total.)

**Permission check hook** (`po()`):

```js
po = () => {
  const { superAdminUser: e } = Ut(i => i.PermissionDetails);
  const t = e?.permission?.permission;
  return {
    isGranted: (module, action) =>
      !Hi.isEmpty(t) && t && Object.keys(t).includes(module) && t[module].includes(action),
    permissions: t,
    superAdminUser: e
  };
};
```

So each staff account's permission record is a map `{ [moduleName]: ["R","C","U","D", ...] }` (letters observed in usage: `R` = Read, `U` = Update; `C`/`D` are strongly implied by the create/delete action wrappers and the "select at least one permission" / "select Read permission first" i18n strings, which describe a checkbox matrix of Read/Create/Update/Delete per module). Staff-management UI text confirms this is presented as a **module × permission-type checkbox grid** ("Please Select Read Permission Before Selecting Other Permissions", "tablePermissions" for per-table column-level sub-permissions).

There's also a `role` field on staff records with at least a `"Support"` role observed in a conditional (`e.role==="Support"`), suggesting a distinguished role name exists alongside the granular permission map — likely `Super Admin` (top-level, ungated) vs `Admin`/sub-admins whose access is governed entirely by the `permission` object above.

Separately, `wV = { ADMIN_EDIT: "admin_edit_key", ADMIN_VIEW: "admin_view_key", DASH_CONFIG: "dashboard_configs" }` are localStorage keys used to persist per-admin UI state (e.g. saved dashboard widget configs, visible-columns state) — not permissions themselves, but namespaced by admin ID.

---

## 5. Domain Enums & Config (from i18n + code constants)

Exact enum *value* strings used on the wire are mostly hidden behind generic action-wrapper calls, but the following were recovered directly from source:

- **KYC statuses / flow states:** `PENDING`, `REJECTED` (approve/verify/reject/activate/inactive actions confirm a state machine: pending → verify/approve → activate, or reject with `rejectDocumentReason`).
- **Tournament status:** `ACTIVE`, `INACTIVE`, `SETTLED`, `CANCELLED` (from `tournamentStatus*` i18n keys and `TOURNAMENT}/{cancel,settlement,toggle}` endpoints).
- **Bonus/wallet transaction types:** `CasinoBonusBet`, `CasinoBonusWin`, `CasinoBonusRefund` (literal enum values found verbatim in source).
- **Bonus categories** (from i18n labels, not raw enum keys): Deposit/Cashback, Free Spins, Loyalty (level-based), Referral, Spin Wheel (daily/weekly/monthly), Coupon-code, Promo-code, Birthday bonus, Cumulative bonus. A dedicated **Wagering Template** entity (min/max wagering limit, wagering multiplier, wagering contribution) is attached to bonuses.
- **RBAC permission letters:** `R`, `U` confirmed in code (`.includes("R")`, `.includes("U")`); `C`/`D` inferred from Create/Delete endpoint coverage per module.
- **Generic transaction/status vocabulary present:** `CANCELLED`, `COMPLETED` (broader Success/Failed/Processing/Pending vocabulary appears only inside human-readable i18n strings, e.g. "Deposit Success Count", "Deposit Failure Count", "Pending Withdrawals", not as raw enum constants — the actual enum values are likely returned by the API and rendered via a status→label lookup not present in this bundle).
- **Notification channels:** in-app `Notifications` panel + `notificationSound`/`notificationType`/`notificationLanguage` selectors; real-time push via socket.io on `/private/suspiciousLogin` and `/private/disputeThreadUpdated`.
- **Chat features:** Channels (global/group), Chat Rain (scheduled prize-drop chat events — daily/weekly/monthly-style promo, with `chatRainPrizemoney`, `chatRainChannelId`), Offensive Word filter (moderation), per-channel member/report views.
- **12 supported UI languages:** EN, DE, ES, FI, FR, IT, JA, KO, NL, PT, RU, ZH.
- **KYC/limits:** self-exclusion, deposit-and-loss limit, betting limit (responsible-gambling controls) are first-class, separately-updatable player attributes.
- **Player tagging & segmentation:** free-form tags plus a segment-builder with "advance filter" (`segment-advance-filter`) — used for targeted bonus/notification campaigns (`referral`, `notify-players`).
- No hardcoded currency-code list (`USD`/`EUR`/etc.) or country-code list was found baked into the bundle — these are fetched at runtime from `settings/currencies`, `settings/countries`, `settings/languages`.

---

## 6. i18n Key Map (English namespace)

The English translation resource (`i18next` `resources.EN`) is a **flat object of 2,114 key → English-string pairs** (not nested namespaces — everything lives at one level, keyed by camelCase or occasionally `snake_case` identifiers, with some duplication e.g. both `wageringCount` and `wagering_count`, suggesting keys were merged from more than one legacy source over time).

Keys were bucketed by substring match against the feature area; counts below (a key can only land in one bucket):

| Feature area | # i18n keys | Sample keys |
|---|---|---|
| Bonus / Loyalty / Wagering | 169 | `bonusTitle`, `wageringMultiplier`, `spinWheelConfiguration`, `loyaltyLevelDetails`, `cumulativeBonusAmount`, `bonusPerformanceInsights` |
| Casino (games/providers/categories) | 136 | `casinoWageredCount`, `topCasinoWagerer`, category/game/provider CRUD labels |
| Payment | 69 | `paymentProvider`, `minimumDeposit`, `withdrawAllowedTooltip`, `FirstTimeDeposit`, `DepositSuccessCount` |
| CMS (pages/banners/emails/gallery) | 58 | `bannerManagement`, `emailTemplateDetailForm`-style labels, gallery upload copy |
| Settings (currency/country/language) | 53 | currency/country toggle & validation copy |
| Player | 45 | player search/detail/comment copy |
| KYC | 43 | `pendingKycRequests`, `manualKycLabels`, `accessDeniedToKycManagement`, `rejectDocumentReason` |
| Chat / Chat Rain / Offensive Words | 41 | `chatRainPrizemoney`, `messageContainsOffensiveWords`, `globalChannel` |
| Sportsbook | 36 | odds/market/league management copy |
| Segment | 25 | segment builder copy |
| Report / Dashboard / KPI | 24 | KPI + dashboard widget titles |
| Tag | 28 | tag CRUD copy |
| Tournament | 27 | `tournamentStatusSettled`, `confirmSettleTournament` |
| Staff / Admin / Roles / Permissions | 20 | `addStaff`, `pleaseSelectAtLeastOnePermission`, `selectReadPermissionFirst` |
| Gamification | 19 | task-based reward copy |
| Notification | 18 | notification type/sound/language selectors |
| Suspicious activity | 18 | risk-flagging copy |
| Referral | 9 | referral program copy |
| Dispute | 1 | `disputeManagement` (dispute UI text mostly lives under generic "ticket"/"reply" wording not caught by the `dispute` substring filter) |
| **Ungrouped (generic UI, validation, common labels)** | 1,275 | date/table/pagination controls, generic form-validation messages ("X is required", "must be a number"), buttons, breadcrumbs |

This confirms the product surface: **Player 360 / KYC / responsible-gambling limits, Bonus & Loyalty engine (with spin-wheel, wagering templates, referrals), Casino game/provider/category catalog management, Sportsbook back-office (odds, markets, leagues), Payment provider & transaction management, CMS (pages/banners/email templates/media gallery), Live chat moderation (channels, chat-rain promos, word filter), Tournament management, Gamification (task-based rewards), Dispute/ticket handling, Segmentation & tagging for targeted campaigns, an AI chatbot admin feature, and a granular per-module RBAC system for staff accounts.**

---

## 7. Security Observations

| Finding | Detail | Risk |
|---|---|---|
| **Hardcoded AES key shipped to browser** | `VITE_APP_FE_ENCRYPTION_KEY = "rb27cry2xn2ysh7823bqxry233x9rn3682323888888q8z90"` is baked into the client bundle and used as `Dfe.AES.encrypt/decrypt` (CryptoJS) key. | Because the key is bundled client-side, anyone can decrypt anything encrypted with it. Its only use found is encrypting the access token before it's written to `localStorage`. This provides **no real security benefit** against an attacker with page/JS access (e.g. XSS) — it only obscures the token from someone glancing at raw localStorage, not from the app's own code or from an attacker who already has code execution. Low real-world impact but worth flagging as security theater; the underlying JWT/access-token is still a bearer credential and its confidentiality relies entirely on this reversible, statically-keyed encryption. |
| **Access token stored in `localStorage`** | `localStorage.setItem("access-token", AES_encrypt(token))`. | localStorage is readable by any JS running on the page (no `httpOnly` protection), so this is vulnerable to token theft via XSS, same as most SPA localStorage-token patterns. The AES wrapping does not mitigate this since the decrypt key ships in the same bundle. |
| **`VITE_APP_AWS_GALLERY_URL` hardcoded** | `https://gammastack-casino.s3.amazonaws.com/` — public S3 bucket used for the CMS/media gallery. | Worth checking bucket ACLs/listing permissions and whether uploaded admin media (banners, KYC-adjacent images, gallery assets) is intended to be public. |
| **Sentry DSN not present in this build** | The env-destructure list `{VITE_APP_API_URL, VITE_APP_FE_ENCRYPTION_KEY, VITE_APP_AWS_GALLERY_URL, VITE_APP_PORT}` omits `VITE_APP_SENTRY_DSN` even though `Sentry.init()` code exists — so in this particular artifact, no DSN ships and error telemetry appears effectively disabled (or is injected via a different runtime mechanism not visible in this static file). | No DSN leak in this build; note for future re-checks if a different build is captured. |
| **Env flags look inconsistent** | The compiled env object shows `MODE:"production", DEV:!0, PROD:!1, SSR:!1` — i.e. `DEV: true` alongside `MODE: "production"`. | Possibly a build artifact/quirk rather than a real dev-mode leak, but combined with the presence of `jsxDEV(...)` calls and `fileName`/`lineNumber` JSX debug annotations throughout the bundle, this strongly suggests **React DevTools debug info and dev-mode React internals are shipped in the "production" build** — increases bundle size and leaks internal file/component structure (source file paths like `/…/Header.jsx` are visible in strings), though it's not a secrets leak per se. |
| **API base URL and full internal route map exposed** | The complete `rn` namespace-to-microservice-path map and all ~250 endpoint paths are recoverable from the client bundle (as demonstrated in Section 2). | Standard for SPAs but worth remembering: the entire admin API surface (including sensitive areas like `internal/create-credentials`, `internal/update-credentials` for third-party integration secrets, and `admin/create-user`) is enumerable by anyone who downloads this JS file, independent of any auth. Access control must be fully enforced server-side per the RBAC model in Section 4 — the client-side `isGranted()` checks are UI-only gating and provide no security boundary. |
| **No CSP/SRI evidence in bundle itself** | Not assessable from the JS alone; would need `index.html`/response headers (out of scope for this file). | — |

---

*Report generated via static string/regex analysis of the minified bundle. Endpoint HTTP methods were verified by tracing the four axios wrapper functions (`Pr`/`Ar`/`W0`/`yg`) back to their `cee.get|post|put|delete` bodies, not merely inferred from naming — this list should be a highly reliable map of the real REST surface. Actual request/response payload shapes were not recovered (would require live traffic capture).*
