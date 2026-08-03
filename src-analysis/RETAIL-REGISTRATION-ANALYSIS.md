# Player Site — Registration, Onboarding and Responsible Gambling

**Target:** `https://white-label-2.gammaplus.io` (the "Arc" tenant)
**Method:** Static analysis of files already on disk under `src-analysis/retail/` — the SSR HTML snapshots captured in a previous session and the 41 downloaded `_next/static` chunks. No page was opened, no form submitted, no request made. Companion to `RETAIL-SOURCE-ANALYSIS.md`; kept as a sibling file rather than a section because it runs long.

`NEXT-STEPS.md` §2 called `/register` "the single most important unseen screen on the player side." It is no longer unseen — the whole form, including the fields, the age gate and the submit path, turned out to be recoverable from the bundle without ever loading the page. The headline is that the form is **smaller and less configurable than expected**, and that is the finding.

---

## 1. Locating the register page

`pages/register.html` (848 KB) is a valid SSR snapshot — `<title>Arc</title>`, HTTP 200, and it loads a route bootstrap:

```
/_next/static/chunks/app/%5Blocale%5D/layout2/register/page-ff9d38fc26603f87.js
```

That file is 216 bytes and does nothing but pull in webpack module `88110`:

```js
(self.webpackChunk_N_E=self.webpackChunk_N_E||[]).push([[5592],{},function(n){
  n.O(0,[1318,2688,1445,9763,8093,2186,6027,4156,231,7007,6823,4340,779,8110,2971,7023,1744],
       function(){return n(n.s=88110)}), _N_E=n.O()
}]);
```

So **the register page is `chunks/8110-f41327b964857623.js`** (webpack modules `88110` → `19918`), and its form-rendering dependency is `chunks/779-547ee61a65012b68.js` (module `70779`, the generic form component). Everything below comes from those two.

**The form is not server-rendered.** `pages/register.html` contains **zero** `<input>`, `<select>` or `<form>` elements — regex-searched, none. The 848 KB is layout chrome plus the RSC flight payload plus the eight-locale i18n tree. The form is a client component that only exists after hydration, which is why the previous session's HTML capture looked empty and why this had to be read out of the chunk instead.

### 1.1 The page also threw a server error at capture time

Being a client component is not the whole explanation. The RSC flight payload carries an **error record** for both the `@auth/register` slot and the `children/register` slot:

```
d:E{"digest":"3178134716"}
12:E{"digest":"3178134716"}
```

The same digest appears in `login.html`. It appears in **neither** `casino.html` nor `support.html`, both of which rendered real content:

| Page | Error digest | Rendered |
|---|---|---|
| `register.html` | `3178134716` | layout only |
| `login.html` | `3178134716` | layout only |
| `bonus.html`, `tournaments.html` | `NEXT_NOT_FOUND` | 404 inside the layout2 shell |
| `casino.html`, `support.html`, `index.html` | none | yes |

So `/register` and `/login` **returned HTTP 200 while their page segment threw server-side** at the moment of capture. Two things follow.

**For method:** an HTTP 200 on this app does not mean the page rendered, and the SSR HTML snapshots cannot be trusted as evidence of what a route contains. The compiled chunks are the reliable source — which is what this analysis used. A future session should check for `digest` records before concluding a route is empty or broken.

**For the vendor:** this is a live defect on the demo tenant, on its two most important conversion pages, and it is consistent with the other backend degradation observed in the same window (`RETAIL-SOURCE-ANALYSIS.md` §7 records the socket endpoint returning 503 and the demo game launch failing to render). Whether the error is transient or permanent cannot be determined from a snapshot — but if a prospective customer evaluates this sandbox, registration and login are what they will try first.

---

## 2. The registration form

### 2.1 Field definitions — verbatim structure

Module `19918` opens with a function `(t, openLegalModal, termsChecked, setTermsChecked) => [ … ]` returning a **literal array of ten field descriptors**. Reformatted, styling classes elided:

| # | `name` | `fieldType` | `inputType` | Label key | Placeholder key | `isRequired` |
|---|---|---|---|---|---|---|
| 1 | `username` | `textField` | `text` | `userName` | `enterUsername` | **true** |
| 2 | `email` | `textField` | `email` | `email` | `enterEmail` | **true** |
| 3 | `password` | `password` | `password` | `password` | `enterPassword` | **true** |
| 4 | `confirmPassword` | `password` | `password` | `confirmPassword` | `confirmPassword` | **true** |
| 5 | `firstName` | `textField` | `text` | `firstName` | `enterFirstName` | **true** |
| 6 | `lastName` | `textField` | `text` | `lastName` | `enterLastName` | **true** |
| 7 | `dateOfBirth` | `date` | `date` | `dateOfBirth` | `profileDob` | **true** |
| 8 | `referralCode` | `textField` | `text` | `referralCode` | `enterReferralCode` | *absent → optional* |
| 9 | `gender` | `radioGroup` | `radio` | `gender` | — | **true** |
| 10 | `termsAndConditions` | `checkboxField` | `checkbox` | — (`checkboxLabel: agreeTermsAndConditions`) | — | **true** |

`gender` is a horizontal radio group with a hardcoded `optionsList`:

```js
optionsList: [
  { key: "male",   value: "male",   label: t("male")   },
  { key: "female", value: "female", label: t("female") },
  { key: "other",  value: "other",  label: t("other")  }
],
orientation: "horizontal"
```

Field 8 is the only one without `isRequired`, making the referral code the single optional input. Field 10 carries layout overrides (`isNewRow: true`, `columnStyle: {gridColumn: "1 / -1", marginTop: "0.5rem"}`) so the consent checkbox spans the full grid width.

### 2.2 What is *not* on the form

Searched module `8110` and the route chunk for each of the following. All absent:

| Expected for a regulated casino | Status |
|---|---|
| **Country / country of residence** | absent — no `country` string anywhere in the register chunk |
| **Currency selection** | absent as a field (but see §2.3 — it is fetched and discarded) |
| **Phone number** | absent — the one `Phone` match is `/windows phone/` inside the user-agent device sniffer |
| **Address / city / postcode** | absent |
| **Bonus / promo code at signup** | absent — no `bonusCode`, no `promoCode`; only the referral code |
| **Affiliate tracking (`btag`, `clickid`, `utm_*`)** | absent |
| **Marketing / promotional opt-in** | absent — one consent checkbox only |
| **Separate age-confirmation checkbox** | absent — age is enforced by the date picker's bounds instead |
| **CAPTCHA** | absent — searched `recaptcha`, `hcaptcha`, `turnstile`, `grecaptcha`, `cf-turnstile`, `altcha` across every chunk, `index.html` and `pages/register.html`: zero matches |
| **CPF / CNPJ (Brazil), or any national ID** | absent from the form, though `cpfError` / `cnpjError` exist in the i18n catalog |

The absence of a country field is the significant one. A country of residence is what a licensed operator uses to apply jurisdiction rules, blocked-market checks and market-specific KYC. This form does not ask for it. *(Inference: geo is presumably resolved server-side from the registration IP — the platform does have IP tooling, `FETCH_IP_LOOKUP_SUCCESS` in the admin Redux actions. That is a reasonable design, but it is an inference; nothing in the client confirms it.)*

The absence of a CAPTCHA on an unauthenticated account-creation endpoint is also worth stating plainly. `RETAIL-SOURCE-ANALYSIS.md` §7.6 noted no CAPTCHA library was bundled; this confirms it specifically for the registration form, which is the endpoint that most needs one. Bot mitigation, if any, is Cloudflare edge-level.

### 2.3 A currency fetch whose result is thrown away

The register component runs this on mount:

```js
const [currencies, setCurrencies] = useState([]);

useEffect(() => {
  (async () => {
    const res = await getCurrencies();                                   // Server Action a1387fae43fd648ee3bb8503cac70e72683f86a2
    if (Array.isArray(res) && res?.data?.currencies.length) setCurrencies(res.data.currencies);
  })();
}, []);
```

`currencies` is **never read again** — it is not passed to `CustomForm`, not rendered, not referenced anywhere in the component's JSX. Observed, not inferred: the identifier appears exactly twice, both in the code above.

So the register page makes a live call to fetch the tenant's currency list on every load and discards the answer. The most economical reading is that a currency selector was designed, built far enough to wire the data source, and then removed or never finished — leaving the fetch behind. Either way, **the player does not choose a currency at signup on this tenant**; the wallet currency must be assigned server-side by default.

### 2.4 Age gate: 18+, hardcoded, client-side only

The `dateOfBirth` field carries computed bounds:

```js
{
  name: "dateOfBirth", fieldType: "date", inputType: "date",
  label: t("dateOfBirth"), placeholder: t("profileDob"), isRequired: true,
  minDate: formatDate(new Date(new Date().setFullYear(new Date().getFullYear() - 200))),
  maxDate: formatDate(new Date(new Date().setFullYear(new Date().getFullYear() - 18)))
}
```

`formatDate` is the `YYYY-MM-DD` helper from module `37672`. So the picker allows dates between *today − 200 years* and *today − 18 years*.

Three observations:

1. **18 is a literal.** It is not fetched, not configurable, not derived from a country. Any jurisdiction with a 21 age limit — several US states, and 21 for casino play in a number of markets — needs a code change and a redeploy. For a white-label platform sold into multiple markets this is a real constraint.
2. **It is a `maxDate` on a date input, i.e. a UI affordance.** It stops the picker offering recent dates; it does not by itself reject a crafted submission. Whether the server enforces the same rule cannot be determined from the client. The i18n catalog has `DOBEarlyThanToday` ("Date Of Birth Must be Earlier than Today") and `dateOfBirthRequired` but **no "must be 18 or over" message** in any of the 1,796 keys — searched. That absence is mild evidence that the age check is only the picker bound, though a server-side rejection could equally surface through a generic error.
3. The 200-year lower bound is cosmetic.

---

## 3. Consent and legal copy

One checkbox covers everything:

```js
{
  id: "register-terms-and-conditions ",                  // note the trailing space in the id
  name: "termsAndConditions", fieldType: "checkboxField", inputType: "checkbox",
  checkboxLabel: t("agreeTermsAndConditions"),           // "I acknowledge and accept the"
  isRequired: true, isNewRow: true,
  columnStyle: { gridColumn: "1 / -1", marginTop: "0.5rem" },
  actionButton: <>
    <button type="button" onClick={() => openLegalModal("terms")}   className="text-l4primary underline">{t("acceptTerms")}</button>
    {" "}{t("and")}
    <button type="button" onClick={() => openLegalModal("privacy")} className="text-l4primary underline ml-1">{t("acceptPrivacyPolicy")}</button>
  </>,
  checked: termsChecked,
  onChange: e => setTermsChecked(e.target.checked)
}
```

Rendering: *"I acknowledge and accept the **Terms and Condition** and **Privacy Policy**"*. A single combined consent — terms and privacy cannot be accepted separately, and there is no distinct marketing consent, so this form does not produce a separable marketing opt-in record. Under GDPR that is the kind of thing a DPO asks about.

The full text of both documents is fetched **from the CMS** rather than shipped:

```js
const slugFor = type => type === "terms"   ? "general-terms"
                      : type === "privacy" ? "privacy-policy"
                      : undefined;

useEffect(() => {
  const slug = slugFor(modalType);
  (async () => {
    if (!slug) return;
    setLoading(true);
    try {
      const res  = await getCmsPage({ slug });                 // Server Action d0639096800e7237d7aff676e11e3270a606c543
      const page = res?.pages?.[0];
      if (page?.content) {
        const localeKey = (language || "en").toUpperCase();    // content is keyed "EN", "FR", …
        setHtml(page.content[localeKey] || "");
      }
    } catch (e) { console.error("Failed to fetch CMS data", e); }
    finally { setLoading(false); }
  })();
}, [modalType, language]);
```

and rendered into a modal through an HTML-string parser (`htmlparser2` is bundled in `chunks/7007-*.js`, and `dangerouslySetInnerHTML` appears across the bundle).

**This is genuinely operator-configurable**, and it is the clearest example of the pattern in the whole flow: legal copy lives in the CMS, keyed by slug and by uppercase locale, editable from the admin's `/cms` screen, per tenant, per language, with no deploy. It is also a stored-HTML → parser path, so the CMS editor is effectively trusted to inject markup into the player site — normal for a CMS, worth knowing.

---

## 4. Validation — Zod, and it runs on the server

The form is rendered by a **generic, config-driven form component** (module `70779`), not by bespoke register JSX:

```js
function CustomForm({
  staticFormFields, leftFormFields, rightFormFields, responsiveFormFields,
  validation, action, INITIAL_STATE, submitLabel, showFieldWiseErrors = false,
  modal = false, extraActionParams = {}, /* … */
}) {
  const [state, formAction] = useFormState((s, fd) => action(s, fd, extraActionParams), INITIAL_STATE);
  const router = useRouter();

  useEffect(() => {
    if (modal && state?.message === "Successfull") router.back();
    if (state?.success && state?.showSnackbar) enqueueSnackbar(t(state?.snackbarMessage || ""), { variant: "success" });
    if (modal && state?.success && state?.isSignUp) router.back();
    if (state?.success && state?.snackbarMessage === "signinSuccessfullyLoggedIn") window.open("/", "_self");
  }, [state]);

  useEffect(() => {
    const url = state?.data?.paymentUrl;
    if (url) window.open(url, "_self");                        // cashier redirect, shared component
  }, [state?.data?.paymentUrl]);

  return (
    <form action={formAction}>
      {/* each field: */}
      <Field {...field} error={state?.zodErrors?.[field.name]} />
      {showFieldWiseErrors && <FieldError error={state?.zodErrors?.[field.name]} />}
      {/* … */}
    </form>
  );
}
```

The decisive detail is `state.zodErrors[field.name]`.

- Validation uses **Zod**. (Fingerprint: `zodErrors` in `chunks/779-*.js`. The `yup` substring matches elsewhere are false positives inside minified identifiers; there is no Formik and no react-hook-form — searched.)
- The errors arrive in the **Server Action's return value**, keyed by field name, alongside `data`, `message`, `success`, `showSnackbar`, `snackbarMessage`, `isSignUp`.
- Therefore **the Zod schema lives in the Server Action, on the server, and is not in the client bundle.** The form is a plain `<form action={serverAction}>` using React's `useFormState`/`useFormStatus` — it posts and waits.

**Consequence for this analysis:** the concrete registration validation rules — password regex, username pattern, length bounds, uniqueness checks — are **not recoverable**. This is the same Server-Action opacity documented in `RETAIL-SOURCE-ANALYSIS.md` §5, and it applies to exactly the thing we most wanted.

It is also a *good* property: server-side validation is authoritative, and there is no client schema to diverge from it. The `validation` prop exists on `CustomForm` and the register page does not pass it.

### 4.1 The rules, inferred from the message catalog

What can be recovered is the **error message vocabulary** from `i18n/en-home-full.json`, which constrains what the server-side schema must be checking. These are messages, not rules — the mapping is inference.

**Password.** A well-defined policy is spelled out:

| Key | Value |
|---|---|
| `passwordTip` | "Password must have at least eight characters with at least one uppercase letter, one lowercase letter, one number and one special character" |
| `passwordMust8Characters` | "Password must be more than 8 characters" |
| `passwordMustContainUppercase` / `...Lowercase` / `...Number` / `...Special` | one message per class |
| `passwordMust20Characters` | "Password must not exceed 20 characters." |
| `passwordMust32Characters` | "Password must be less than 32 characters" |
| `matchPassword` / `passwordDon'tMatch` | confirm-field equality |

Note the **three mutually inconsistent maximum lengths** in one catalog: 20, 32, and — from the sign-in and reset flows — 16 (`signinErrorsPasswordMaxLength`, `resetPasswordNewPasswordErrorsMaxLength`: "Maximum 16 characters allowed"). `resetPasswordNewPasswordErrorsPattern` states "Password should be 8 to 16 alphanumeric and special characters." Whatever the register schema enforces, the catalog cannot tell us which of 16/20/32 it is, and the three flows may genuinely differ — a user could set a 20-character password at signup and be unable to re-enter it at a 16-character-capped reset form. Flagging as a likely real inconsistency, not a confirmed one.

**Username.** `signinErrorsUsernamePattern`: "Username can only contain letters, numbers, and underscores, and must start with a letter"; `signinErrorsUserNamePattern`: "only letters and numbers are allowed" — again two different statements. Min 4 (`signinErrorsUserNameMinLength`) or min 3 (`profileErrorsUserNameMinLength`), max 20.

**Email.** `invalidEmail`, `emailRequired`, `emailAlreadyRegistered` / `emailIsTaken` (server-side uniqueness), `emailNotVerified`, `profileErrorsEmailMaxLength` "Maximum 20 characters allowed in email" — a 20-character cap on an email address would reject a great many real addresses. Present in the profile-edit vocabulary; whether registration applies it is unknown.

**Names, gender, DOB.** `firstNameRequired`, `lastNameRequired`, `profileErrorsFirstNameTooLong` (50), `genderRequired`, `selectvalidGenderValue`, `dateOfBirthRequired`, `DOBEarlyThanToday`.

**Referral.** `invalidReferralCode`, `referralCodeIsInvalid`, `InvalidReferralCodeErrorType`, `referralInactive`, `referralLimitExceeded` — so the code is validated server-side, can be inactive, and has a usage cap.

---

## 5. Submit path and post-registration flow

### 5.1 The action

```js
<CustomForm
  responsiveFormFields={buildRegisterFields(t, type => { setModalType(type); setShowTermsModal(true); }, termsChecked, setTermsChecked)}
  isSubmit
  submitLabel={t("register")}
  action={registerAction}
  INITIAL_STATE={{ data: null }}
  formClass="responsive-customizable-column-form"
  modal
  backDropParams
  extraActionParams={{
    device:     getDeviceType(userAgent).toLocaleUpperCase(),                        // iPhone|iPod|Android|Windows Phone|iPad|Tablet|Mobile|Desktop|Unknown
    deviceInfo: `${uaParsed.name.toLocaleUpperCase()},${uaParsed.os.toLocaleUpperCase()}`
  }}
/>
```

Registration is a **Next.js Server Action**, id `828a9405ab89e8be3a6ead3d27df295065600f25` (module `96077`, export `XH`). No REST path reaches the browser. The browser POSTs to the register URL itself with a `Next-Action` header; the call to `white-label-api.gammaplus.io` happens inside the Next.js runtime.

`userAgent` is a **prop passed from the server** — the device fingerprint is derived from the server-observed UA and submitted as `device` + `deviceInfo` alongside the form data. That matches the admin's device-history features (`FETCH_USER_DEVICE_HISTORY_SUCCESS`, `FETCH_PLAYERS_LOGIN_DEVICE_AND_SESSION_SUCCESS`): the platform records the signup device from the first request.

`modal: true` means registration renders in a dialog over the previous route and `router.back()` on success — `/register` is a page but the product treats signup as an overlay.

### 5.2 Related Server Action ids

Module `96077` (auth) and `98332` (CMS/reference data), with call sites where traceable:

| Export | Action id | Purpose |
|---|---|---|
| `XH` | `828a9405ab89e8be3a6ead3d27df295065600f25` | **Register** |
| `ZP` (default) | `3fa9ad69fede2e3830c5703490510daaebf8b54f` | Login *(inferred — default export of the auth action module)* |
| `Rv` | `accda4ac90813ee10d260fb835c31537c1bcfb2e` | **Resend OTP** — called as `resendOtp(userEmail, "login")` |
| `b7` | `8131490e7210c9ee3843a07efe371152977b7056` | auth-related, call site not traced |
| `xf` | `ed14be9b2f17e2bc96f4a75fdbeb82c9ade82f11` | auth-related, call site not traced |
| — | `cca3b6a1…`, `22877e6b…`, `d4075a07…`, `b37a8115…`, `cf090302…` | declared but unexported |
| `jQ` | `a1387fae43fd648ee3bb8503cac70e72683f86a2` | **Get currencies** (the discarded call, §2.3) |
| `tX` | `d0639096800e7237d7aff676e11e3270a606c543` | **Get CMS page by slug** (terms/privacy, §3) |
| `ZP` | `96991a803cf7e9ac41c832ee17d2860040de9640` | CMS default export |

### 5.3 What happens on success

From `CustomForm`'s effect and the i18n catalog:

- `state.isSignUp && state.success` → `router.back()`, closing the modal. There is **no auto-login**.
- Email verification is required and is the gate on withdrawals, not on play: `sentAVerification` ("We've sent a verification link"), `checkYourEmail`, `accountVerified`, `emailVerify`, `resendEmail`, `accAlreadyVerified`, and pointedly `notVerified` — *"You must verify your email address to withdraw money"*, plus `pleaseVerifyFirstText` ("In order to be able to request a withdrawal, we first need to verify your email address…"). So a player can register and presumably deposit/play before verifying; the hard stop is at cashout.
- A separate **OTP** path exists (`EnterLoginOtp`, `LoginwithOtp`, `enterOTP`, `invalidOtp`, `otpVerified`, `resendOtp`) with a live `ResendOtp` component calling `resendOtp(userEmail, "login")`. The `"login"` context argument implies other contexts exist. 2FA enrolment copy is present too (`otp_setup_title`, `otp_setup_description`, `select_2fa_method`, `otp_description`: "sent to your phone via SMS, email, or app").
- KYC is post-registration, not at signup: `kycVerification`, `kycVerificationProcessText`, `ManualKYC`, `UseShuftiProtoCompleteYourKYCVerificationInstantly` (**Shufti Pro** as the identity vendor), `generateKYCLink`.

**Onboarding sequence, as far as it can be reconstructed:** register (10 fields) → email verification link → optional OTP/2FA enrolment → play → KYC required before withdrawal. Deliberately low-friction at the top of the funnel with compliance deferred to the cashout, which is the standard commercial pattern and is exactly the design a regulator will probe.

---

## 6. Operator-configurable vs hardcoded — the core question

This was the point of the exercise: the admin exposes `GET_REGISTRATION_FIELDS` / `UPDATE_REGISTRATION_FIELDS` (`ADMIN-BUNDLE-ANALYSIS.md` §2, Settings/Config) and a `/form-fields` screen that silently redirected to the dashboard when the previous session tried to open it. Does the player-side register form consume that configuration?

**On the evidence in the client bundle: no.**

What is observed:

1. The field list is a **literal array in a function body** in `chunks/8110-*.js`. `isRequired` is a literal `true`; `optionsList` for gender is literal; the ten fields are in fixed order.
2. **Nothing fetches a field configuration.** The only network call the register component makes on mount is `getCurrencies()`, and it discards the result (§2.3). There is no `formFields`, `registrationFields`, `fieldName`, `isActive` or `fieldType`-keyed payload fetched or read.
3. `pages/register.html`'s RSC flight payload contains no field-configuration data — the form is client-only and the array is compiled in.

What complicates it, and belongs in the answer:

4. **The machinery for a configurable form exists and is deliberately unused here.** `CustomForm` takes `staticFormFields`, `leftFormFields`, `rightFormFields` and `responsiveFormFields` as props, `.map()`s over them, honours a per-field **`isHide`** flag (`!field?.isHide && <div className="input-box …">`), and accepts a `validation` prop. Every hook needed to drive the form from server-supplied config is present. The register page simply hands it a constant and omits `validation`.
5. **Validation is server-side (§4), so the client cannot tell us what the server enforces.** It is entirely possible the operator's registration-field settings are applied *inside* the register Server Action — controlling which fields are required or rejected — while the rendered form stays fixed. That would give an operator control over validation but not over what the player sees.

**Both readings, stated honestly:**

- *Reading A (favoured by the evidence):* the `layout2` register page is hardcoded and ignores the admin's registration-field configuration. The configuration capability exists in the platform but this skin does not consume it — consistent with `layout2` being one of several shipped front-end skins (`RETAIL-SOURCE-ANALYSIS.md` §2), where a different skin might.
- *Reading B (cannot be excluded):* the admin config is enforced server-side in the Server Action, and the client form is a fixed superset. This would be invisible from the client.

What can be said without qualification is that **an operator cannot add, remove or reorder a visible registration field on this tenant without a front-end code change and redeploy.** For a white-label product where "configurable registration fields" is a standard checklist item, that is the finding, and it is a meaningful gap between what the admin API advertises and what the player front end does.

Contrast with what *is* genuinely per-tenant: legal copy via CMS slug (§3), currency list via API (fetched, if unused), theme/skin via `layout1`/`layout2`, and whole-page enable/disable (§8). The registration *fields* are the outlier.

---

## 7. Responsible gambling

`/responsible-gambling` 404s on this tenant (§8), so nothing is observable in rendered form. But the i18n catalog carries **93 RG-related keys**, and they describe a fairly complete toolset. This is the strongest available evidence of what the platform can do — reported as capability, not as something seen working.

### 7.1 Limit types and periods

Four limit types × three periods, plus session controls:

| Limit | Description string | Periods |
|---|---|---|
| **Deposit** | "Once activated, it sets the maximum amount of money a player can add to their account within a specified period." | daily / weekly / monthly |
| **Loss** | "…the maximum amount a player can lose during a designated timeframe or across various games." | daily / weekly / monthly |
| **Wager** | "…the maximum amount a player can bet on a single game or during a specific period." | daily / weekly / monthly |
| **Session** | "Once activated, this will alert you after you have been logged in for a specific amount of time." | single value |
| **Self-exclusion** | "…you will no longer be able to play at GS Casino, nor you will have access to your account." | preset periods |
| **Take a Break** | "You are about to block the access to your account for a preset period of time." | preset periods |

Per-period, per-type keys exist in full: `{daily,weekly,monthly}{Deposit,Loss,Wager}Limit{Required,MinError,MaxError}`. Enforcement messages appear on the play side too — `dailyBetLimitExceeded`, `weeklyBetLimitExceeded`, `monthlyBetLimitExceeded`, `DailyDepositLimitExceeded` — so limits are checked at bet and deposit time, not merely stored.

### 7.2 The cooling-off rule, which is the regulator-relevant part

```
limit24Reset  = "Once you set Wager, Loss, Deposit limits then it will be editable and removeable
                 after 24hrs but limits can be decreased immediately."
limitSet24Hrs = "You are about to set the following limit to your account. Please note, that in case
                 you want to change the limit, you can do that after 24hrs."
sessionLimitInfo = "…in case you want to change the limit, it will take 24 hours for the limit to …"
```

**Decreases take effect immediately; increases and removals are delayed 24 hours.** That asymmetry is the standard responsible-gambling requirement and the single most important RG detail to have recovered — it is what a regulator asks about first, and the platform implements it. A 24-hour delay is on the short side; several jurisdictions require longer (MGA and UKGC-style regimes commonly expect 24h minimum with cooling-off on increases, some markets more), so this is a "meets the common baseline, check per market" rather than a "meets everything".

Also present: `pleaseEnsureDailyLessThanWeeklyLessThanMonthlyValuesForLimits` — daily ≤ weekly ≤ monthly is enforced as a cross-field rule.

### 7.3 Exclusion states and other

`excludedTemporarily`, `excludedPermanently` ("Excluded permanently please contact provider"), `selfExclusion`, `selfExclusionLimitInfo`, `takeABreak`, `pleaseWaitUntilSessionTimeHasElapsed`, `limitCantSetBefore`, `limitRemoved`, `limitNotFound`, `limitValueExceedMaximum`, `maxRetryLimitExceed`. Plus per-game limits surfaced from the provider (`GameLimitsMinimumBet`, `GameLimitsMaximumBet`, `GameLimitsMaxWinFor1Bet`) and a `profileSideNavigatorGameLimits` nav entry.

**Not found** (searched): any *reality check* / periodic play-time popup, and any *affordability* or source-of-funds check. `sessionInfo` describes an alert after a logged-in duration, which is adjacent to a reality check but is framed as a one-shot session limit.

The player-side limits UI corresponds to the admin's player-detail **Limits** tab, which the previous session did see (`FINDINGS.md`), so both halves of this feature exist.

---

## 8. The six pages that 404, and what the catalog says they would contain

`vip.html`, `promotions.html`, `providers.html`, `live-casino.html`, `responsible-gambling.html`, `all.html` are each ~4.5 KB. **All six are pure Next.js 404 shells — no content whatsoever.** Confirmed structurally, not by size:

```html
<html id="__next_error__"> … <meta name="robots" content="noindex"/> …
"6:[\"NotFound\",\"vip\",\"c\"]"                       ← route segment resolves to NotFound
"[\"$\",\"title\",null,{\"children\":\"404: This page could not be found.\"}]"
```

Stripping tags and scripts yields the empty string for every one. They differ from each other only in the slug embedded in the flight payload. **Nothing about these features is recoverable from these files** — anyone reading the capture directory should not mistake their 4.5 KB for content.

The features nonetheless exist in the shared codebase, and the i18n catalog describes them:

**VIP / loyalty — 78 keys.** A points-based ladder: *"For every €10 in cash bets placed at our casino, you'll earn 1 loyalty point"*, climbing `loyaltyLevels` with `levelUpPoints`, a `loyaltyTableHeading` "Levels and Rewards Overview" with columns Loyalty Level / Loyalty Points / **Daily Cashback**, and `cashbackUp` "Cashback up to 25%". Reaching **Level 6** unlocks the VIP programme proper, whose eleven advertised benefits are enumerated: exclusive bonuses, higher deposit/withdrawal limits, personal account manager, faster withdrawals, monthly cashback, event/tournament invitations, birthday bonuses, luxury gifts. Jackpot tiers `levelMini` / `levelMinor` / `levelMajor` / `levelGrand`. `loyaltyPerCurrency` implies per-currency point configuration — matching the admin's `CREATE_LOYALTY_LEVEL_SUCCESS` / `GET_LOYALTY_LEVEL_SUCCESS` actions. Marketing copy still says "Gammastack" and "GS Casino" rather than "Arc" — more unscrubbed vendor branding, same class as the `Tower.bet` leak.

**Promotions — 59 keys.** Coupon/promo-code redemption (`enterCouponCode`, `couponCodeApplied`, `couponCodeAlreadyClaimed`, `invalidPromoCode`), a bonus lifecycle (`promotionsActivatebtn` ACTIVATE → `promotionsClaimBtn` CLAIM → `promotionsCancelBtn`), bonus categories `promotionsAvailabilityTabDeposit` / `promotionsAvailabilityTabLosing`, and a two-step deposit-bonus flow ("Activate Bonus code" then "Deposit amount to wallet"). Note this is where promo codes live — **at redemption, not at registration** (§2.2).

Both confirm `RETAIL-SOURCE-ANALYSIS.md` §7.7: GammaStack feature-flags entire routes per tenant, and the "Arc" tenant has VIP, promotions, providers, live-casino and the RG page switched off while the code and the full translation catalog for all of them still ship to every visitor.

---

## 9. Observations

1. **The registration form is thinner than the platform behind it.** Ten fields, no country, no currency, no phone, no CAPTCHA, one combined consent. The admin exposes registration-field configuration; this front end does not consume it (§6). An operator entering a market with different data-collection requirements needs a code change.

2. **The 18+ age gate is a hardcoded date-picker bound.** No jurisdiction-driven age, no 21 option, and no "must be over 18" message anywhere in 1,796 keys — so there is no evidence of a distinct server-side age rejection, though its absence from the catalog is not proof of its absence from the server (§2.4).

3. **No CAPTCHA on account creation**, confirmed by direct search across every chunk. Account-creation abuse (bonus farming, multi-accounting) is a live commercial problem for casinos, and the platform's own answer to it appears to be detection after the fact — the admin has `GET_DUPLICATE_USERS_SUCCESS`, device history, IP lookup and suspicious-activity tooling — rather than prevention at the door.

4. **The password policy contradicts itself across flows**: max 20 (register copy), max 32 (another key), max 16 (sign-in and reset). If those caps are real, a password valid at signup may be unusable at reset (§4.1).

5. **A live API call on every register page load whose result is discarded** (§2.3). Minor, but it is a per-pageview call to the backend on an unauthenticated high-traffic page.

6. **Legal consent is a single checkbox** covering terms and privacy together, with no separable marketing opt-in (§3).

7. **Responsible gambling is the strongest part of the picture**: four limit types across three periods, session limits, self-exclusion, take-a-break, bet-time and deposit-time enforcement, daily ≤ weekly ≤ monthly consistency, and the correct decrease-immediately / increase-after-24h asymmetry (§7). The gap versus a strict regime is the absence of any reality-check or affordability tooling.

8. **Validation being server-side is a genuine strength** — and simultaneously the reason the concrete rules could not be recovered. The same Server Action opacity that protects the REST contract from this analysis also protects it from a would-be attacker enumerating the schema.

---

## Appendix: where each finding came from

| Finding | File |
|---|---|
| Register page = chunk 8110 | `retail/chunks/app/[locale]/layout2/register/page-ff9d38fc26603f87.js` |
| Field array, age bounds, gender options, consent checkbox, terms/privacy CMS fetch, currency fetch, submit wiring | `retail/chunks/8110-f41327b964857623.js` (modules `19918`, `96077`, `98332`, `37672`, `19720`) |
| Generic form component, `zodErrors`, `useFormState`, success handling, `isHide` | `retail/chunks/779-547ee61a65012b68.js` (module `70779`) |
| No SSR form; page is client-rendered | `retail/pages/register.html` |
| Six routes are Next.js 404 shells | `retail/pages/{vip,promotions,providers,live-casino,responsible-gambling,all}.html` |
| Password/username/email rules, RG limits and 24h rule, VIP ladder, promotions, OTP/KYC/email-verification vocabulary | `retail/i18n/en-home-full.json` (1,796 keys) |
| `htmlparser2` / `dangerouslySetInnerHTML` | `retail/chunks/7007-242a9fbe34fcd3ce.js`, `6931-*.js`, `fd9d1056-*.js` |
